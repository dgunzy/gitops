# Cabot Cup next deployment

This is a staged deployment target for the unified Go application. It has no
`HTTPRoute` and is scaled to zero, so it cannot receive production traffic.

AWS Secrets Manager must contain `k0s/cabot-cup/app` with:

- `OIDC_ISSUER_URL`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, and
  `OIDC_REDIRECT_URL` for the provider-neutral Google OIDC integration.
- `SESSION_SECRET` and `CSRF_SECRET`, generated independently for this application.

Database credentials are intentionally separate and arrive through the
`cabot-cup-db` Secret. Add S3 configuration only after selecting the upload bucket
and defining least-privileged application access; do not reuse backup credentials.

Before activation:

1. Confirm the Cabot ImageRepository, ImagePolicy, and ImageUpdateAutomation are
   ready and resolve the current immutable CI-produced tag and digest.
2. Verify the database migration job and both ExternalSecrets.
3. Confirm `/livez` and `/readyz` exist in that image.
4. Scale the Deployment and test it without changing the production routes.
