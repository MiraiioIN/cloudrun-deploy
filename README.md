# Deploy to Google Cloud Run

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Deploy%20to%20Cloud%20Run-blue?logo=github)](https://github.com/marketplace/actions/deploy-to-google-cloud-run-with-env-support)
[![License](https://img.shields.io/badge/license-Commercial-green)](#license)

A GitHub Action that builds, pushes, and deploys your application to **Google Cloud Run** with first-class support for environment variables. Your `.env` file is passed as a **Base32-encoded GitHub Secret**, decoded at deploy time, and converted to Cloud Run's native env-vars YAML format — no secrets ever touch your repo.

## Why Base32?

GitHub Secrets have constraints around encoding. Base32 is safe for storage in GitHub Secrets — it's ASCII-only, has no special shell characters, and round-trips perfectly, unlike raw `.env` contents which can break on quotes, newlines, and multiline values.

## Features

- **Secure env handling** — `.env` contents never appear in your repo or build logs
- **Artifact Registry** — builds and pushes your Docker image to Google Artifact Registry
- **Automatic env conversion** — `.env` is parsed and converted to Cloud Run YAML env-vars format
- **GCP SA key injection** — optionally injects the service account key (base64) as `GCP_SA_KEY_B64`
- **Cleanup** — all sensitive intermediate files are removed after deploy (even on failure)
- **Configurable** — supports custom Dockerfile paths, extra `gcloud run deploy` flags, and auth controls

## Quick Start

### 1. Encode your `.env` file

```bash
base32 < .env
```

Copy the output and store it as a GitHub Secret named `ENV_FILE_BASE32`.

### 2. Store your GCP service account key

Save the raw JSON key as a GitHub Secret named `GCP_SA_KEY`.

### 3. Add the workflow

```yaml
name: Deploy to Cloud Run
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Cloud Run
        uses: MiraiioIN/cloudrun-deploy@v1
        id: deploy
        with:
          service_name: my-api
          region: us-west1
          project_id: my-gcp-project
          image_repository: my-repo
          env_file_base32: ${{ secrets.ENV_FILE_BASE32 }}
          gcp_sa_key: ${{ secrets.GCP_SA_KEY }}

      - name: Print URL
        run: echo "Deployed to ${{ steps.deploy.outputs.service_url }}"
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `service_name` | **Yes** | — | Cloud Run service name |
| `region` | **Yes** | — | GCP region (e.g. `us-west1`) |
| `project_id` | **Yes** | — | GCP project ID |
| `image_repository` | **Yes** | — | Artifact Registry repository name |
| `env_file_base32` | **Yes** | — | Base32-encoded `.env` file contents |
| `gcp_sa_key` | **Yes** | — | GCP service account JSON key (raw JSON) |
| `backend_path` | No | `backend` | Path to directory containing your `Dockerfile` |
| `allow_unauthenticated` | No | `true` | Allow public access to the deployed service |
| `cloud_run_flags` | No | `""` | Additional flags for `gcloud run deploy` |

## Outputs

| Output | Description |
|---|---|
| `service_url` | The URL of the deployed Cloud Run service |
| `image_uri` | Full URI of the pushed container image in Artifact Registry |

## Prerequisites

Before using this action, ensure you have:

1. **A GCP project** with billing enabled
2. **Cloud Run API** enabled (`gcloud services enable run.googleapis.com`)
3. **Artifact Registry API** enabled (`gcloud services enable artifactregistry.googleapis.com`)
4. **An Artifact Registry Docker repository** created:
   ```bash
   gcloud artifacts repositories create my-repo \
     --repository-format=docker \
     --location=us-west1
   ```
5. **A service account** with the following roles:
   - `roles/run.admin`
   - `roles/iam.serviceAccountUser`
   - `roles/artifactregistry.writer`
   - `roles/storage.admin` (for pushing images)
6. **A Dockerfile** in your backend directory

## Advanced Usage

### Custom Dockerfile location

```yaml
- uses: MiraiioIN/cloudrun-deploy@v1
  with:
    service_name: my-api
    region: us-central1
    project_id: my-gcp-project
    image_repository: my-repo
    env_file_base32: ${{ secrets.ENV_FILE_BASE32 }}
    gcp_sa_key: ${{ secrets.GCP_SA_KEY }}
    backend_path: src/server
```

### Restrict access & set resource limits

```yaml
- uses: MiraiioIN/cloudrun-deploy@v1
  with:
    service_name: my-api
    region: us-east1
    project_id: my-gcp-project
    image_repository: my-repo
    env_file_base32: ${{ secrets.ENV_FILE_BASE32 }}
    gcp_sa_key: ${{ secrets.GCP_SA_KEY }}
    allow_unauthenticated: "false"
    cloud_run_flags: "--memory=1Gi --cpu=2 --min-instances=1 --max-instances=10"
```

### Multiple environments

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: MiraiioIN/cloudrun-deploy@v1
        with:
          service_name: my-api-staging
          region: us-west1
          project_id: my-gcp-project-staging
          image_repository: my-repo
          env_file_base32: ${{ secrets.ENV_FILE_BASE32_STAGING }}
          gcp_sa_key: ${{ secrets.GCP_SA_KEY_STAGING }}

  deploy-production:
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: MiraiioIN/cloudrun-deploy@v1
        with:
          service_name: my-api-production
          region: us-west1
          project_id: my-gcp-project-prod
          image_repository: my-repo
          env_file_base32: ${{ secrets.ENV_FILE_BASE32_PROD }}
          gcp_sa_key: ${{ secrets.GCP_SA_KEY_PROD }}
```

## How It Works

```
┌─────────────────────────────────────────────────────────┐
│  GitHub Secret: ENV_FILE_BASE32                         │
│  (base32-encoded .env contents)                         │
└──────────────────────┬──────────────────────────────────┘
                       │ base32 --decode
                       ▼
┌─────────────────────────────────────────────────────────┐
│  .env file (decoded in runner, never committed)         │
│  DB_HOST=my-db.example.com                              │
│  API_KEY=sk_live_abc123                                 │
└──────────────────────┬──────────────────────────────────┘
                       │ env-to-yaml.js
                       ▼
┌─────────────────────────────────────────────────────────┐
│  env.yaml (Cloud Run native format)                     │
│  DB_HOST: "my-db.example.com"                           │
│  API_KEY: "sk_live_abc123"                              │
│  GCP_SA_KEY_B64: "eyJ0eXBlIjoi..."                     │
└──────────────────────┬──────────────────────────────────┘
                       │ gcloud run deploy --env-vars-file
                       ▼
┌─────────────────────────────────────────────────────────┐
│  Cloud Run service with env vars injected               │
│  All intermediate files cleaned up                      │
└─────────────────────────────────────────────────────────┘
```

## Security

- Secrets are passed via environment variables, never as command-line arguments
- All intermediate files (`env.decoded`, `env.yaml`, `gcp_sa_b64.txt`, `.env`) are removed after deployment — even if the deploy fails
- The `.env` file is never committed to your repository
- See [SECURITY.md](SECURITY.md) for our security policy

## Encoding Reference

**Encode your .env file (macOS/Linux):**

```bash
base32 < .env
```

**Verify it decodes correctly:**

```bash
echo "YOUR_BASE32_STRING" | base32 -d
```

**On macOS with coreutils:**

```bash
brew install coreutils
gbase32 < .env
```

## Troubleshooting

| Problem | Solution |
|---|---|
| `base32: invalid input` | Ensure the secret has no trailing whitespace or newlines. Re-encode with `base32 < .env \| tr -d '\n'`. |
| `Permission denied` on Artifact Registry | Grant `roles/artifactregistry.writer` to your service account. |
| `Cloud Run deploy fails` | Ensure Cloud Run and Artifact Registry APIs are enabled. Check `gcloud services list`. |
| `Docker build fails` | Verify `backend_path` points to the directory containing your `Dockerfile`. |

## License

This is a **commercial** GitHub Action. See [LICENSE](LICENSE) for terms.

Copyright (c) 2026. All rights reserved.
