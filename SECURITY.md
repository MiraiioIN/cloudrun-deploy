# Security Policy

## How This Action Handles Secrets

This action is designed with security as a core concern:

1. **No secrets in logs** — All secret values are passed via environment variables, never as command-line arguments (which can appear in process listings and logs).

2. **Intermediate file cleanup** — Files containing decoded secrets (`env.decoded`, `env.yaml`, `gcp_sa_b64.txt`, and the copied `.env`) are deleted after every run, including on failure, via an `if: always()` cleanup step.

3. **No network exfiltration** — This action only communicates with Google Cloud APIs. It does not phone home or transmit data to any third-party service.

4. **Base32 encoding** — `.env` contents are Base32-encoded before storage in GitHub Secrets, ensuring safe round-tripping without shell interpretation issues.

## Supported Versions

| Version | Supported |
|---------|-----------|
| v1.x    | Yes       |

## Reporting a Vulnerability

If you discover a security vulnerability in this action, please report it responsibly:

1. **Do NOT** open a public GitHub issue
2. Email **security@miraiio.org** with:
   - A description of the vulnerability
   - Steps to reproduce
   - Potential impact
3. You will receive a response within **48 hours**
4. We will work with you to understand and address the issue before any public disclosure

## Best Practices for Users

- **Rotate secrets regularly** — Update your `GCP_SA_KEY` and `ENV_FILE_BASE32` secrets periodically
- **Principle of least privilege** — Grant the service account only the roles listed in the README
- **Audit access** — Review who has access to your repository secrets
- **Pin action versions** — Use a specific tag (e.g., `@v1.0.0`) rather than `@main` to avoid supply chain attacks
