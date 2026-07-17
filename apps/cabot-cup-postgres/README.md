# Cabot Cup PostgreSQL foundation

This base adds the `cabot_cup` database to the existing CNPG cluster. The login
and its password secret are reconciled with the shared cluster foundation so
Flux can make the managed role ready before creating the database.
It does not create another PostgreSQL cluster and does not grant access to the
Masters Pool application namespace.

Before Flux reconciles this base, AWS Secrets Manager must contain
`k0s/cabot-cup/postgres` with:

- `password`: a generated password for the fixed PostgreSQL role `cabot_cup`.
- `DATABASE_URL`: a PostgreSQL URL for `cabot_cup` using the same password and
  `masters-pool-postgres-rw.databases.svc.cluster.local:5432/cabot_cup` with
  `sslmode=require`.

The `ClusterExternalSecret` exposes only `DATABASE_URL`, and only in namespaces
labelled `db.cabot-cup/access: "true"`. The same label grants network access to
port 5432. The `Database` uses reclaim policy `retain`, so removing this base
does not drop financial data.

## Test database rehearsal (`DATABASE_MODE=test`)

`database-test.yaml` adds a disposable `cabot_cup_test` database owned by the
same `cabot_cup` role. The application selects it by setting
`DATABASE_MODE=test`, which reads `TEST_DATABASE_URL` instead of
`DATABASE_URL`; the two URLs must differ or the app refuses to start.

Before merging, add `TEST_DATABASE_URL` to `k0s/cabot-cup/postgres` in AWS
Secrets Manager (same host/role/password, database `cabot_cup_test`,
`sslmode=require`). A missing property fails the entire ExternalSecret sync.

The app rejects `DATABASE_MODE=test` when `APP_ENV=production`, so rehearsal
runs use an acceptance instance with `APP_ENV=staging`. The real production
deployment cannot be silently flipped onto test data.
