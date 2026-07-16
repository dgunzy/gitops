# Cabot Cup migration 000001

This release gate runs the Cabot image's embedded migration command:

```text
/app/cabot-cup migrate
```

The image must treat already-applied migrations as success. Flux image automation
updates this Job and the web Deployment together; `force: true` on the migration
Kustomization replaces the immutable completed Job so the gate is evaluated for
the new image before application reconciliation.

The Job runs in `databases`, receives only `DATABASE_URL` from its dedicated
`cabot-cup-migrate-db` ExternalSecret, and is admitted to CNPG by its exact pod
labels. It cannot receive ingress and can egress only to DNS and TCP 5432. A failed
Job blocks the public Deployment rollout and remains available for inspection.
