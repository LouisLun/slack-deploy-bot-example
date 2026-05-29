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
| `DOCKERHUB_USERNAME` | `coffeesouffle` | Docker Hub username (public image, no auth needed) |

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
