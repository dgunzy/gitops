# Cabot Cup acceptance environment

This package serves `test.cabotcup.ca` from a separate `cabot-cup-test`
namespace. Both the migration Job and web Deployment set `APP_ENV=staging` and
`DATABASE_MODE=test`, so every application read/write and outbox consumer uses
`TEST_DATABASE_URL` (`cabot_cup_test`). Production remains pinned to
`DATABASE_MODE=real`.

The acceptance environment deliberately uses the normal production-built image
and Google OIDC. It does not contain the build-tagged `/dev/login` bypass. After
the first migration, bootstrap one real Google email as the test owner with the
application's `bootstrap-owner` command. Do not store that email in Git; provide
it only to the one-time controlled Job. Once signed in, the owner can invite
other acceptance testers through the normal UI.

The test hostname sends `X-Cabot-Environment: acceptance` and
`X-Robots-Tag: noindex, nofollow`. The database is disposable, but reset remains
an explicit operator action: verify `DATABASE_MODE=test` and the selected
database name before clearing any data.

Release order is enforced by Flux:

1. `cabot-cup-postgres` provisions and verifies `cabot_cup_test`.
2. `cabot-cup-test-migrate` runs the schema/public snapshot migration against
   `TEST_DATABASE_URL`.
3. `cabot-cup-test` releases the web Deployment and HTTPS route only after the
   migration and platform certificate/Gateway configuration are healthy.
