# slack-deploy-bot-example

GitHub Actions workflow for deploying [slack-deploy-bot](https://hub.docker.com/repository/docker/coffeesouffle/slack-deploy-bot) to Google Cloud Run.

## Required Secrets & Variables

### Secrets

| Name | Description |
|---|---|
| `WIF_PROVIDER` | Workload Identity Federation provider resource name |
| `WIF_SERVICE_ACCOUNT` | Service account email used for deployment |
| `DEPLOY_CONFIG_JSON` | (optional) Full deploy config JSON string — overrides GCS when set |
| `SLACK_SIGNING_SECRET` | Slack App signing secret |
| `SLACK_BOT_TOKEN` | Slack Bot User OAuth Token (`xoxb-…`) |
| `GH_CLIENT_ID` | GitHub OAuth App client ID |
| `GH_CLIENT_SECRET` | GitHub OAuth App client secret |

### Variables

| Name | Example | Description |
|---|---|---|
| `GCP_REGION` | `asia-east1` | Cloud Run region |
| `CLOUD_RUN_SERVICE` | `slack-deploy-bot` | Cloud Run service name |
| `GCS_BUCKET_NAME` | `my-bucket` | GCS bucket storing the deploy config |
| `GCS_CONFIG_FILE_PATH` | `deploy-config.json` | Config file path inside the bucket |

## Setup

### 1. Use this template

Click **Use this template** on GitHub to create your own repo.

### 2. Create Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**.
2. Name it (e.g. `deploy-bot`) and pick your workspace.

**Configure OAuth Scopes:**

3. **OAuth & Permissions** → **Bot Token Scopes** → Add:
   - `chat:write` — post messages
   - `commands` — receive slash commands

**Add Slash Commands** (use placeholder URL for now, update in step 8):

4. **Slash Commands** → **Create New Command** for each:

   | Command | Request URL | Description |
   |---|---|---|
   | `/deploy` | `https://YOUR_CLOUD_RUN_URL/slack/events` | Deploy a release group — usage: `/deploy <group> <release title>` |
   | `/hotfix` | `https://YOUR_CLOUD_RUN_URL/slack/events` | Hotfix a single project — usage: `/hotfix <project> <release title>` |

**Install & get tokens:**

5. **Install App** → **Install to Workspace** → Authorize.
6. Copy `SLACK_SIGNING_SECRET` → **Basic Information** → App Credentials → Signing Secret.
7. Copy `SLACK_BOT_TOKEN` → **OAuth & Permissions** → Bot User OAuth Token (`xoxb-…`).

### 3. Create GitHub OAuth App

1. Go to **GitHub → Settings → Developer settings → OAuth Apps → New OAuth App**.
2. Fill in the fields (use placeholder URL for now, update in step 8):

   | Field | Value |
   |---|---|
   | Application name | `slack-deploy-bot` (or any name) |
   | Homepage URL | `https://YOUR_CLOUD_RUN_URL` |
   | Authorization callback URL | `https://YOUR_CLOUD_RUN_URL/auth/github/callback` |

3. Copy `GH_CLIENT_ID` → **Client ID** (shown on app page).
4. Copy `GH_CLIENT_SECRET` → click **Generate a new client secret** → copy immediately (shown once only).

> The bot authenticates users via GitHub OAuth to trigger workflows. The authenticated user must have **write access** to the target repos.

### 4. Set up Google Cloud

**Create a service account:**
```bash
gcloud iam service-accounts create github-actions \
  --display-name="GitHub Actions"
```

**Grant roles:**
```bash
# Required to deploy and update Cloud Run services
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:github-actions@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/run.admin"

# Required to act as the Cloud Run runtime service account
# Replace RUNTIME_SA with your Cloud Run runtime service account
# (default: PROJECT_NUMBER-compute@developer.gserviceaccount.com)
gcloud iam service-accounts add-iam-policy-binding \
  RUNTIME_SA@PROJECT_ID.iam.gserviceaccount.com \
  --member="serviceAccount:github-actions@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# Required if using GCS config (Option A in step 5)
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:github-actions@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

**Set up Workload Identity Federation:**
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

**Bind the service account to your repo:**
```bash
gcloud iam service-accounts add-iam-policy-binding \
  github-actions@PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github/attribute.repository/OWNER/REPO"
```

**Get values for secrets:**
- `WIF_PROVIDER` → `projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github/providers/github-provider`
- `WIF_SERVICE_ACCOUNT` → `github-actions@PROJECT_ID.iam.gserviceaccount.com`

### 5. Prepare deploy-config.json

**Option A — GCS (recommended):**
```bash
gcloud storage buckets create gs://MY_BUCKET --location=asia-east1
gcloud storage cp deploy-config.json gs://MY_BUCKET/deploy-config.json
```

**Option B — Secret:**
Set `DEPLOY_CONFIG_JSON` to the full JSON string. GCS variables are ignored when this secret is set.

See [deploy-config.json Format](#deploy-configjson-format) for the full schema.

### 6. Set GitHub Secrets & Variables

Go to **repo → Settings → Secrets and variables → Actions**.

**Secrets:**

| Secret | Value |
|---|---|
| `WIF_PROVIDER` | From step 4 |
| `WIF_SERVICE_ACCOUNT` | From step 4 |
| `SLACK_SIGNING_SECRET` | From step 2 |
| `SLACK_BOT_TOKEN` | From step 2 |
| `GH_CLIENT_ID` | From step 3 |
| `GH_CLIENT_SECRET` | From step 3 |
| `DEPLOY_CONFIG_JSON` | (optional) From step 5 Option B |

**Variables:**

| Variable | Value |
|---|---|
| `GCP_REGION` | e.g. `asia-east1` |
| `CLOUD_RUN_SERVICE` | e.g. `slack-deploy-bot` |
| `GCS_BUCKET_NAME` | From step 5 Option A (not needed for Option B) |
| `GCS_CONFIG_FILE_PATH` | e.g. `deploy-config.json` (not needed for Option B) |

### 7. Deploy

Go to **Actions → Deploy to Cloud Run → Run workflow**, optionally specify an image tag (default: `latest`).

After the workflow completes, get the Cloud Run service URL:
```bash
gcloud run services describe $CLOUD_RUN_SERVICE \
  --region=$GCP_REGION \
  --format='value(status.url)'
```

### 8. Update Slack & GitHub OAuth URLs

Replace `YOUR_CLOUD_RUN_URL` placeholders from steps 2 and 3 with the actual URL:

- **Slack** → [api.slack.com/apps](https://api.slack.com/apps) → your app → **Slash Commands** → edit each command → update Request URL.
- **GitHub OAuth App** → **Settings → Developer settings → OAuth Apps** → your app → update Homepage URL and Authorization callback URL.

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
          { "name": "console",  "repo": "myorg/console",  "workflows": ["release-cd.yml"] },
          { "name": "frontend", "repo": "myorg/frontend", "mergeOnly": true }
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
    "website":  { "repo": "myorg/website",  "workflows": ["release-cd.yml"] },
    "frontend": { "repo": "myorg/frontend", "mergeOnly": true }
  }
}
```

- `groups` — used by `/deploy <group-name> <release title>`, defines step order and parallel projects
- `projects` — used by `/hotfix <project-name> <release title>`, flat map of project name → repo + workflows
- `mergeOnly: true` — merge the PR and stop; no tag, no workflow trigger, no GitHub Release. Use for projects that auto-deploy on merge (e.g. Cloudflare Pages)

## Deploy Flow (`/deploy`)

```
/deploy production Fix checkout flow
  │
  ├─ Step 1 (all projects concurrent)
  │   ├─ restful:  release-cd.yml ─────────────► wait ──► release "Fix checkout flow"
  │   ├─ wms:      release-cd.yml ──► wait ──┐
  │   │            notify.yml     ──► wait ──┴─► release "Fix checkout flow"
  │   ├─ console:  release-cd.yml ─────────────► wait ──► release "Fix checkout flow"
  │   └─ frontend: merge PR ──► (done, auto-deploys via Cloudflare)
  │
  └─ Step 2 (starts only after Step 1 fully completes)
      └─ website:  release-cd.yml ─────────────► wait ──► release "Fix checkout flow"
```

- Projects **within the same step** are triggered in parallel.
- Workflows **within the same project** are also triggered in parallel (all fire simultaneously, wait for all to complete).
- `mergeOnly` projects: merge PR only — no tag, no workflow, no release.
- The provided release title is used as the GitHub Release name.
- A failed workflow aborts the release for that project and blocks the next step.
- Projects with no open PR labelled `production` (case-insensitive) are **skipped** and reported in Slack.

## Hotfix Flow (`/hotfix`)

```
/hotfix <project-name> <release title>
```

1. Looks up `<project-name>` in `config.projects`.
2. Finds the most recently updated open PR labelled `hotfix` (case-insensitive).
3. Merges the PR.
4. If `mergeOnly: true` — stops here (auto-deploys externally).
5. Creates a version tag on the merge commit.
6. Triggers all of that project's workflows in parallel, waits for all to complete.
7. Creates a GitHub Release with the provided release title on success.

Example:
```
/hotfix wms Fix order sync bug
```

## Cloud Run Deploy Workflow

Triggered manually via **GitHub → Actions → Deploy to Cloud Run → Run workflow**.

Optionally specify an image tag to deploy (defaults to `latest`).
