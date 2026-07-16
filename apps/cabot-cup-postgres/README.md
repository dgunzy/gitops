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
