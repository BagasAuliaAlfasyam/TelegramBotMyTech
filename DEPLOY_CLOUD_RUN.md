# Deploying the Telegram bot to Cloud Run

This project runs as a long‑running polling bot. The recommended target on Google Cloud is **Cloud Run (fully managed)**. The container only needs outbound internet access to reach Telegram, Google Sheets, and your S3 endpoint.

## Prerequisites
- gcloud CLI authenticated to your project (`gcloud auth login` / `gcloud config set project <PROJECT_ID>`).
- Enable services once: `gcloud services enable run.googleapis.com secretmanager.googleapis.com`
- A Google service account JSON that has access to the target spreadsheet.

## Build container image
```bash
cd TelegramBot
gcloud builds submit --tag gcr.io/PROJECT_ID/telegram-bot
```

## Prepare secrets and env vars
Save your service account JSON to Secret Manager (contents only, not the filename):
```bash
gcloud secrets create GOOGLE_SA_JSON --data-file=white-set-293710-9cca41a1afd6.json
```

Required environment variables (see `config.py`):
- `TELEGRAM_BOT_TOKEN_COLLECTING`
- `TELEGRAM_BOT_TOKEN_REPORTING`
- `GOOGLE_SERVICE_ACCOUNT_JSON` (path to the JSON file inside the container)
- `GOOGLE_SPREADSHEET_NAME`
- `GOOGLE_WORKSHEET_NAME`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `AWS_S3_BUCKET`
- `AWS_S3_ENDPOINT`

Optional:
- `TARGET_GROUP_COLLECTING`, `TARGET_GROUP_REPORTING`
- `AWS_S3_PUBLIC_BASE_URL`, `AWS_S3_REGION`
- `AWS_S3_MEDIA_PREFIX` (default `tech-media/`)
- `AWS_S3_SIGNATURE_VERSION` (`s3` or `s3v4`, default `s3`)
- `AWS_S3_ADDRESSING_STYLE` (`virtual` or `path`, default `virtual`)

## Deploy to Cloud Run
Example deployment (adjust REGION, PROJECT_ID, env vars, and Secret names):
```bash
SERVICE=telegram-bot
REGION=asia-southeast2

gcloud run deploy $SERVICE \
  --image gcr.io/PROJECT_ID/telegram-bot \
  --region $REGION \
  --platform managed \
  --max-instances 1 \
  --allow-unauthenticated \
  --set-env-vars TELEGRAM_BOT_TOKEN_COLLECTING=xxx,TELEGRAM_BOT_TOKEN_REPORTING=yyy,GOOGLE_SPREADSHEET_NAME=\"Your Sheet\",GOOGLE_WORKSHEET_NAME=\"Sheet1\",AWS_ACCESS_KEY_ID=...,AWS_SECRET_ACCESS_KEY=...,AWS_S3_BUCKET=...,AWS_S3_ENDPOINT=https://... \
  --set-env-vars AWS_S3_PUBLIC_BASE_URL=https://...,AWS_S3_REGION=ap-southeast-1 \
  --set-env-vars TARGET_GROUP_COLLECTING=123456789,TARGET_GROUP_REPORTING=123456789 \
  --set-env-vars GOOGLE_SERVICE_ACCOUNT_JSON=/var/secrets/google_sa.json \
  --mount=type=secret,source=GOOGLE_SA_JSON,target=/var/secrets/google_sa.json
```

Notes:
- Cloud Run `--mount` stores the secret as a file at the given path so `GOOGLE_SERVICE_ACCOUNT_JSON` can point to it.
- If you want to keep Telegram tokens in Secret Manager too, add more `--set-secrets` flags or additional mounts instead of plain `--set-env-vars`.
- `--max-instances 1` avoids duplicated polling, but you can raise it if needed; the Telegram API tolerates multiple polling clients but may duplicate work.

## Health check
The bot exposes `/health` (and `/ping`) commands. From any allowed chat, send `/health` to verify the container is running.
