# Transcribing large sessions with AWS S3 (+ optional webhook)

Deepgram's synchronous `v1/listen` endpoint holds the HTTP connection open for
the entire transcription. For a long session (e.g. ~3 hr / 200+ MB) with a
slower model (`diarize_model=latest`), that connection can outlive Deepgram's
processing window and come back as a **`Gateway Timeout`** — no transcript.

This guide moves the heavy lifting to AWS in two tiers:

- **Tier 1** hosts the audio in **S3** and hands Deepgram a URL, so you never
  upload the large file from your laptop. Still a normal synchronous request —
  enough for `diarize_model=v1` (see the finding in [`README.md`](README.md),
  section 2).
- **Tier 2** adds an async **webhook**: Deepgram returns immediately and later
  `POST`s the finished JSON to a tiny **Lambda**, which writes it to S3. Use this
  when a job is slow enough to time out synchronously (e.g. `latest` on a
  multi-hour file).

> **Why Lambda and not just S3 for the webhook?** Plain S3 can't be Deepgram's
> callback target — Deepgram `POST`s a raw JSON body, while S3 presigned URLs
> expect a `PUT` (or a multipart-form `POST`). A one-function Lambda catches the
> `POST` and stores it.

All commands use placeholders — substitute your own bucket name (S3 bucket names
are globally unique) and region. The Deepgram key is the `$DEEPGRAM_API_KEY` you
already load from `.env` (`set -a; source .env; set +a`).

---

## 0. Prerequisites

- An AWS account.
- The **AWS CLI** installed and configured once:

  ```bash
  aws configure          # access key, secret key, default region (e.g. us-east-1)
  aws sts get-caller-identity   # confirms your credentials work
  ```

This AWS setup is **optional infrastructure** — only worth it for large/slow
jobs. A direct `v1` upload (per the README) usually finishes fine without any of
this.

Throughout, these are the placeholders to replace:

| Placeholder        | Example                        |
|--------------------|--------------------------------|
| `<BUCKET>`         | `<yourname>-deepgram`          |
| `<REGION>`         | `us-east-1`                    |
| `<PRESIGNED_URL>`  | output of `aws s3 presign` (below) |
| `<FUNCTION_URL>`   | output of the Function URL step (Tier 2) |

---

## Tier 1 — S3-hosted audio, synchronous request

### 1. Create a private bucket

Default S3 settings keep the bucket **private** (all public access blocked),
which is what we want — Deepgram reaches the object through a temporary presigned
URL, not public hosting.

```bash
aws s3 mb s3://<BUCKET> --region <REGION>
```

### 2. Upload the audio

```bash
aws s3 cp session.m4a s3://<BUCKET>/
```

### 3. Generate a presigned GET URL

A presigned URL is a temporary, expiring link to the private object — no public
access required.

```bash
aws s3 presign s3://<BUCKET>/session.m4a --expires-in 3600
```

The URL only has to stay valid long enough for Deepgram to **start** fetching
(seconds), so 1 hour (`3600`) is plenty. The SigV4 maximum is 7 days
(`604800`). Copy the printed URL for the next step.

### 4. Submit to Deepgram by URL

Send a small JSON body pointing at the audio instead of streaming the bytes —
Deepgram fetches the file itself, so nothing large leaves your machine.

```bash
curl --request POST \
  --url 'https://api.deepgram.com/v1/listen?model=nova-3&diarize_model=v1&punctuate=true&smart_format=true&utterances=true' \
  --header "Authorization: Token $DEEPGRAM_API_KEY" \
  --header 'Content-Type: application/json' \
  --data '{"url":"<PRESIGNED_URL>"}' \
  > session.deepgram.json
```

This is still synchronous — the transcript comes back on this request and lands
in `session.deepgram.json`. Keep `diarize_model=v1` per the README finding; only
reach for `latest` (and Tier 2) if `v1` is visibly mis-splitting speakers.

### 5. Clean up (optional)

```bash
aws s3 rm s3://<BUCKET>/session.m4a
```

---

## Tier 2 — Async webhook (S3 + Lambda)

Use this when the job is slow enough to time out synchronously. Deepgram returns
a `request_id` immediately, processes in the background, then `POST`s the result
to a public **Lambda Function URL**, which writes it to S3.

Do Tier 1 steps 1–3 first (you still host the audio in S3 and get a
`<PRESIGNED_URL>`). Then set up the receiver once:

### 1. IAM role for the Lambda

Create a role the Lambda assumes, with permission to write results to the bucket.

```bash
# Trust policy: allow Lambda to assume the role
cat > trust.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
JSON

aws iam create-role --role-name deepgram-callback-role \
  --assume-role-policy-document file://trust.json

# CloudWatch Logs permissions
aws iam attach-role-policy --role-name deepgram-callback-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Permission to write the transcript into s3://<BUCKET>/results/
cat > s3put.json <<'JSON'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow",
 "Action":"s3:PutObject","Resource":"arn:aws:s3:::<BUCKET>/results/*"}]}
JSON

aws iam put-role-policy --role-name deepgram-callback-role \
  --policy-name write-results --policy-document file://s3put.json
```

### 2. The Lambda function

The handler decodes the POST body (Function URLs may base64-encode it) and writes
it to S3. Save as `lambda_function.py`:

```python
import base64, os, boto3

s3 = boto3.client("s3")
BUCKET = os.environ["RESULT_BUCKET"]

def handler(event, context):
    # Optional shared-secret check (see step 4); no-op if TOKEN isn't set.
    expected = os.environ.get("TOKEN")
    if expected:
        got = (event.get("queryStringParameters") or {}).get("token")
        if got != expected:
            return {"statusCode": 403, "body": "forbidden"}

    body = event.get("body", "") or ""
    body = base64.b64decode(body) if event.get("isBase64Encoded") else body.encode()
    s3.put_object(
        Bucket=BUCKET,
        Key="results/session.deepgram.json",
        Body=body,
        ContentType="application/json",
    )
    return {"statusCode": 200, "body": "ok"}
```

Package and create the function (use the role ARN printed in step 1):

```bash
zip function.zip lambda_function.py

aws lambda create-function --function-name deepgram-callback \
  --runtime python3.12 --handler lambda_function.handler \
  --role arn:aws:iam::<ACCOUNT_ID>:role/deepgram-callback-role \
  --timeout 30 --zip-file fileb://function.zip \
  --environment "Variables={RESULT_BUCKET=<BUCKET>}"
```

### 3. Add a public Function URL

This gives the Lambda a plain HTTPS endpoint Deepgram can POST to.

```bash
aws lambda create-function-url-config \
  --function-name deepgram-callback --auth-type NONE

# Allow anonymous invokes of the Function URL
aws lambda add-permission --function-name deepgram-callback \
  --statement-id fnurl --action lambda:InvokeFunctionUrl \
  --principal '*' --function-url-auth-type NONE
```

The first command prints a `FunctionUrl` like
`https://<id>.lambda-url.<REGION>.on.aws/` — that's your `<FUNCTION_URL>`.

### 4. Optional — shared-secret hardening

The Function URL is public, so anyone who learns it could POST junk that
overwrites your result. To lock it down, set a secret and require it:

```bash
aws lambda update-function-configuration --function-name deepgram-callback \
  --environment "Variables={RESULT_BUCKET=<BUCKET>,TOKEN=<SECRET>}"
```

Then append `?token=<SECRET>` to the callback URL in step 5. The handler above
already rejects mismatches with `403`.

### 5. Submit with a callback

```bash
curl --request POST \
  --url 'https://api.deepgram.com/v1/listen?model=nova-3&diarize_model=latest&punctuate=true&smart_format=true&utterances=true&callback=<FUNCTION_URL>' \
  --header "Authorization: Token $DEEPGRAM_API_KEY" \
  --header 'Content-Type: application/json' \
  --data '{"url":"<PRESIGNED_URL>"}'
```

You get back `{"request_id":"..."}` right away (not a transcript). Deepgram
processes in the background — a multi-hour file can take many minutes — then
POSTs the finished JSON to your Lambda.

> If your callback URL carries a query string (e.g. `?token=<SECRET>`),
> URL-encode the whole `<FUNCTION_URL>` value when placing it after `&callback=`
> so the outer Deepgram URL parses correctly.

### 6. Fetch the result

```bash
aws s3 cp s3://<BUCKET>/results/session.deepgram.json .
```

Then clean up the audio (`aws s3 rm s3://<BUCKET>/session.m4a`); the transcript
is small and cheap to keep.

---

## Which tier do I need?

| Situation                                             | Use     |
|-------------------------------------------------------|---------|
| `diarize_model=v1`, any normal-length session         | Tier 1  |
| Uploading from the laptop is slow / flaky             | Tier 1  |
| `diarize_model=latest` or job times out synchronously | Tier 2  |
| Very long recording (multi-hour)                      | Tier 2  |

Once you have a good `session.deepgram.json`, continue with **step 3 (Format the
transcript)** in [`README.md`](README.md) — or just run the **build-speaker-mapping**
and **translate-deepgram** skills, which do that step for you.
