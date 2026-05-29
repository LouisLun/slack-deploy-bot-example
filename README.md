# slack-deploy-bot-example

GitHub Actions workflow for deploying [slack-deploy-bot](https://hub.docker.com/repository/docker/coffeesouffle/slack-deploy-bot) to Google Cloud Run.

## Usage

Use this repo as a GitHub template to create your own repo, then configure the required secrets and variables.

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

## How to Obtain Each Secret

### WIF_PROVIDER & WIF_SERVICE_ACCOUNT (Google Cloud)

1. Create a service account:
   ```bash
   gcloud iam service-accounts create github-actions \
     --display-name="GitHub Actions"
   ```
2. Grant it the `Cloud Run Developer` role on your project. Also grant `Storage Object Viewer` if using the GCS config provider.
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

### SLACK_SIGNING_SECRET & SLACK_BOT_TOKEN (Slack App)

#### Create the App

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**.
2. Name it (e.g. `deploy-bot`) and pick your workspace.

#### Configure OAuth Scopes

3. **OAuth & Permissions** → **Bot Token Scopes** → Add:
   - `chat:write` — post messages
   - `commands` — receive slash commands

#### Add Slash Commands

4. **Slash Commands** → **Create New Command** for each:

   | Command | Request URL | Description |
   |---|---|---|
   | `/deploy` | `https://YOUR_CLOUD_RUN_URL/slack/events` | Deploy a release group |
   | `/hotfix` | `https://YOUR_CLOUD_RUN_URL/slack/events` | Hotfix a single project |

   > Replace `YOUR_CLOUD_RUN_URL` after deploying (step 5 in Setup).

#### Install & Get Tokens

5. **Install App** → **Install to Workspace** → Authorize.
6. `SLACK_SIGNING_SECRET` → **Basic Information** → App Credentials → Signing Secret.
7. `SLACK_BOT_TOKEN` → **OAuth & Permissions** → Bot User OAuth Token (`xoxb-…`).

### GITHUB_CLIENT_ID & GITHUB_CLIENT_SECRET (GitHub OAuth App)

1. Go to **GitHub → Settings → Developer settings → OAuth Apps → New OAuth App**.
2. Set **Authorization callback URL** to your Cloud Run service URL + `/auth/github/callback`.
3. After creating: `GITHUB_CLIENT_ID` → Client ID, `GITHUB_CLIENT_SECRET` → generate and copy a Client Secret.

## Setup

### 1. Fork or use this template

Click **Use this template** on GitHub to create your own repo.

### 2. Prepare deploy-config.json

Either upload to GCS (recommended) or set as a secret (see format below).

**Option A — GCS:**
```bash
# Create bucket
gcloud storage buckets create gs://MY_BUCKET --location=asia-east1

# Upload config
gcloud storage cp deploy-config.json gs://MY_BUCKET/deploy-config.json
```

Grant the service account read access:
```bash
gcloud storage buckets add-iam-policy-binding gs://MY_BUCKET \
  --member="serviceAccount:github-actions@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

**Option B — Secret:**
Set `DEPLOY_CONFIG_JSON` secret to the full JSON string. GCS variables are ignored when this secret is set.

### 3. Set GitHub Secrets

Go to **repo → Settings → Secrets and variables → Actions → Secrets**:

| Secret | Value |
|---|---|
| `WIF_PROVIDER` | See [WIF setup](#wif_provider--wif_service_account-google-cloud) |
| `WIF_SERVICE_ACCOUNT` | See [WIF setup](#wif_provider--wif_service_account-google-cloud) |
| `SLACK_SIGNING_SECRET` | From Slack App Basic Information |
| `SLACK_BOT_TOKEN` | From Slack App OAuth & Permissions |
| `GITHUB_CLIENT_ID` | From GitHub OAuth App |
| `GITHUB_CLIENT_SECRET` | From GitHub OAuth App |
| `DEPLOY_CONFIG_JSON` | (optional) Full config JSON string |

### 4. Set GitHub Variables

Go to **repo → Settings → Secrets and variables → Actions → Variables**:

| Variable | Value |
|---|---|
| `GCP_REGION` | e.g. `asia-east1` |
| `CLOUD_RUN_SERVICE` | e.g. `slack-deploy-bot` |
| `GCS_BUCKET_NAME` | GCS bucket name from step 2 |
| `GCS_CONFIG_FILE_PATH` | e.g. `deploy-config.json` |

### 5. Deploy

Go to **Actions → Deploy to Cloud Run → Run workflow**, optionally specify an image tag (default: `latest`).

The workflow authenticates via WIF, then deploys the public Docker image to Cloud Run with the configured environment variables.

## deploy-config.json Format

Set as `DEPLOY_CONFIG_JSON` secret (inline JSON) or upload to GCS. Full format:

```json
{
  "groups": {
    "production": [
      {
        "step": 1,
        "projects": [
          { "name": "restful",  "repo": "myorg/restful",  "workflows": ["release-cd.yml"] },
          { "name": "wms",      "repo": "myorg/wms",      "workflows": ["release-cd.yml", "notify.yml"] },
          { "name": "console",  "repo": "myorg/console",  "workflows": ["release-cd.yml"] }
        ]
      },
      {
        "step": 2,
        "projects": [
          { "name": "website",  "repo": "myorg/website",  "workflows": ["release-cd.yml"] }
        ]
      }
    ]
  },
  "projects": {
    "restful":  { "repo": "myorg/restful",  "workflows": ["release-cd.yml"] },
    "wms":      { "repo": "myorg/wms",      "workflows": ["release-cd.yml", "notify.yml"] },
    "console":  { "repo": "myorg/console",  "workflows": ["release-cd.yml"] },
    "website":  { "repo": "myorg/website",  "workflows": ["release-cd.yml"] }
  }
}
```

- `groups` — used by `/deploy <group-name>`, defines step order and parallel projects
- `projects` — used by `/hotfix <project-name>`, flat map of project name → repo + workflows

## Deploy Flow (`/deploy`)

```
/deploy production
  │
  ├─ Step 1 (all projects concurrent)
  │   ├─ restful:  release-cd.yml ─────────────► wait ──► release
  │   ├─ wms:      release-cd.yml ──► wait ──┐
  │   │            notify.yml     ──► wait ──┴─► release
  │   └─ console:  release-cd.yml ─────────────► wait ──► release
  │
  └─ Step 2 (starts only after Step 1 fully completes)
      └─ website:  release-cd.yml ─────────────► wait ──► release
```

- Projects **within the same step** are triggered in parallel.
- Workflows **within the same project** are also triggered in parallel (all fire simultaneously, wait for all to complete).
- A failed workflow aborts the release for that project and blocks the next step.
- Projects with no open PR labelled `production` (case-insensitive) are **skipped** and reported in Slack.

## Hotfix Flow (`/hotfix`)

```
/hotfix <project-name>
```

1. Looks up `<project-name>` in `config.projects`.
2. Finds the most recently updated open PR labelled `hotfix` (case-insensitive).
3. Triggers all of that project's workflows in parallel, waits for all to complete.
4. Creates a GitHub Release on success.

Example:
```
/hotfix wms
```

## Cloud Run Deploy Workflow

Triggered manually via **GitHub → Actions → Deploy to Cloud Run → Run workflow**.

Optionally specify an image tag to deploy (defaults to `latest`).
