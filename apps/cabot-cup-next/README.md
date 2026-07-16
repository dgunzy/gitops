# Cabot Cup unified deployment

This is the public deployment for the unified Go application. Its `HTTPRoute` owns
`cabotcup.ca`; authenticated and administrative routes must remain unavailable until
their authorization implementation is complete.

AWS Secrets Manager must contain `k0s/cabot-cup/app` with:

- `OIDC_ISSUER_URL`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, and
  `OIDC_REDIRECT_URL` for the provider-neutral Google OIDC integration.
- `SESSION_SECRET` and `CSRF_SECRET`, generated independently for this application.

Database credentials are intentionally separate and arrive through the
`cabot-cup-db` Secret. Add S3 configuration only after selecting the upload bucket
and defining least-privileged application access; do not reuse backup credentials.

Release ordering:

1. Confirm the Cabot ImageRepository, ImagePolicy, and ImageUpdateAutomation are
   ready and resolve the current immutable CI-produced tag and digest.
2. Flux reconciles `cabot-cup-postgres`, then the versioned migration Job, then this
   Deployment and route.
3. Verify the migration Job and both ExternalSecrets before promoting a new image.
4. Confirm `/livez` and `/readyz` and the public routes after rollout.

The legacy `cabot-cup` Deployment and Service remain in the cluster for rollback,
but its conflicting `HTTPRoute` is intentionally absent.
