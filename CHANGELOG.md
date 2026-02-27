# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-02-27

### Added

- Initial release
- Build and push Docker image to Google Artifact Registry
- Decode Base32-encoded `.env` file and convert to Cloud Run YAML env-vars format
- Authenticate with GCP via service account JSON key
- Deploy to Cloud Run with full env var support
- Inject `GCP_SA_KEY_B64` as an environment variable for runtime GCP access
- Automatic cleanup of all sensitive intermediate files (even on failure)
- Configurable `allow_unauthenticated` flag
- Support for additional `gcloud run deploy` flags via `cloud_run_flags`
- Output `service_url` and `image_uri` for downstream steps
