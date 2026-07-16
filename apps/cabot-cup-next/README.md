# Cabot Cup unified deployment

This is the public deployment for the unified Go application. Its `HTTPRoute` owns
`cabotcup.ca`; authenticated and administrative routes must remain unavailable until
their authorization implementation is complete.

The public-only Deployment intentionally receives no database, OIDC, session, or S3
credentials. AWS Secrets Manager may hold `k0s/cabot-cup/app` with:

- `OIDC_ISSUER_URL`, `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`, and
  `OIDC_REDIRECT_URL` for the provider-neutral Google OIDC integration.
- `SESSION_SECRET` and `CSRF_SECRET`, generated independently for this application.

Database credentials remain separately stored in `k0s/cabot-cup/postgres`. Add both
secret distributions only when authenticated handlers are implemented and tested;
do not reuse backup credentials.

Release ordering:

1. Confirm the Cabot ImageRepository, ImagePolicy, and ImageUpdateAutomation are
   ready and resolve the current immutable CI-produced tag and digest.
2. Flux reconciles `cabot-cup-postgres`, then the versioned migration Job, then this
   Deployment and route.
3. Verify the migration Job before promoting a new image.
4. Confirm `/livez` and `/readyz` and the public routes after rollout.

The legacy `cabot-cup` Deployment and Service remain in the cluster for rollback,
but its conflicting `HTTPRoute` is intentionally absent.
