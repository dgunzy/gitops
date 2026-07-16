# Cabot Cup migration 000001

This release gate runs the Cabot image's embedded migration command:

```text
/app/cabot-cup migrate
```

The image must treat already-applied migrations as success. Flux image automation
updates this Job and the web Deployment together; `force: true` on the migration
Kustomization replaces the immutable completed Job so the gate is evaluated for
the new image before application reconciliation.

The Job receives only `DATABASE_URL` from `cabot-cup-db`, cannot receive ingress,
and can egress only to cluster DNS and the shared CNPG cluster. A failed Job blocks
the public Deployment rollout and remains available for operator inspection.
