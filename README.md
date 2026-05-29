# slack-deploy-bot-example

GitHub Actions workflow example for deploying [slack-deploy-bot](https://github.com/LouisLun/slack-deploy-bot) to Google Cloud Run.

## Usage

Copy `.github/workflows/deploy-cloudrun.yml` into your slack-deploy-bot repo and configure the required secrets and variables.

## Required Secrets & Variables

### Secrets

| Name | Description |
|---|---|
| `WIF_PROVIDER` | Workload Identity Federation provider resource name |
| `WIF_SERVICE_ACCOUNT` | Service account email used for deployment |
| `DEPLOY_CONFIG_JSON` | (optional) Full deploy config JSON string — overrides GCS when set |
| `SLACK_SIGNING_SECRET` | Slack App signing secret |
| `SLACK_BOT_TOKEN` | Slack Bot User OAuth Token (`xoxb-…`) |
| `GITHUB_CLIENT_ID` | GitHub OAuth App client ID |
| `GITHUB_CLIENT_SECRET` | GitHub OAuth App client secret |

### Variables

| Name | Example | Description |
|---|---|---|
| `GCP_REGION` | `asia-east1` | Cloud Run region |
| `CLOUD_RUN_SERVICE` | `slack-deploy-bot` | Cloud Run service name |
| `GCS_BUCKET_NAME` | `my-bucket` | GCS bucket storing the deploy config |
| `GCS_CONFIG_FILE_PATH` | `deploy-config.json` | Config file path inside the bucket |
| `DOCKERHUB_USERNAME` | `louislun` | Docker Hub username (public image, no auth needed) |

## How to Obtain Each Secret

### WIF_PROVIDER & WIF_SERVICE_ACCOUNT (Google Cloud)

1. Create a service account:
   ```bash
   gcloud iam service-accounts create github-actions \
     --display-name="GitHub Actions"
   ```
2. Grant it the `Cloud Run Developer` and `Storage Object Viewer` roles on your project.
3. Enable Workload Identity Federation:
   ```bash
   gcloud iam workload-identity-pools create github \
     --location="global" \
     --display-name="GitHub Actions Pool"

   gcloud iam workload-identity-pools providers create-oidc github-provider \
     --location="global" \
     --workload-identity-pool="github" \
     --issuer-uri="https://token.actions.githubusercontent.com" \
     --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"
   ```
4. Bind the service account to your repo:
   ```bash
   gcloud iam service-accounts add-iam-policy-binding \
     github-actions@PROJECT_ID.iam.gserviceaccount.com \
     --role="roles/iam.workloadIdentityUser" \
     --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github/attribute.repository/OWNER/REPO"
   ```
5. `WIF_PROVIDER` → full provider resource name, e.g. `projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github/providers/github-provider`
6. `WIF_SERVICE_ACCOUNT` → service account email, e.g. `github-actions@PROJECT_ID.iam.gserviceaccount.com`

### DOCKERHUB_USERNAME (Docker Hub)

The image `DOCKERHUB_USERNAME/slack-deploy-bot` is a public image — no login required. Set this variable to the Docker Hub username that owns the image.

### SLACK_SIGNING_SECRET & SLACK_BOT_TOKEN (Slack App)

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → select your app (or create one).
2. `SLACK_SIGNING_SECRET` → **Basic Information** → App Credentials → Signing Secret.
3. `SLACK_BOT_TOKEN` → **OAuth & Permissions** → OAuth Tokens → Bot User OAuth Token (`xoxb-…`).
   - Required scopes: `chat:write`, `commands` (add more as needed by slack-deploy-bot).

### GITHUB_CLIENT_ID & GITHUB_CLIENT_SECRET (GitHub OAuth App)

1. Go to **GitHub → Settings → Developer settings → OAuth Apps → New OAuth App**.
2. Set **Authorization callback URL** to your Cloud Run service URL + `/auth/github/callback`.
3. After creating: `GITHUB_CLIENT_ID` → Client ID, `GITHUB_CLIENT_SECRET` → generate and copy a Client Secret.

## Workflow

Triggered manually via **GitHub → Actions → Deploy to Cloud Run → Run workflow**.

Optionally specify an image tag to deploy (defaults to `latest`).
