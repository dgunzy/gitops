# CloudNativePG (vendored Helm output)

The operator is installed as **rendered (`helm template`) manifests**, committed to git
and versioned by operator release in a sub-directory (e.g. `1.29.1/`). CRDs are split
out from the operator so Flux can apply them in the correct order.

```
1.29.1/                 # CNPG operator (app) version
  crds/                 #   10 CustomResourceDefinitions  -> Flux Kustomization: cnpg-crds
  operator/             #   controller + RBAC + webhooks  -> Flux Kustomization: cnpg-operator (dependsOn cnpg-crds)
```

Ordering is enforced in `clusters/k0s-cluster-1/flux-sync.yaml`:
`cnpg-crds` (wait for CRDs Established) → `cnpg-operator` (wait for Deployment Ready)
→ `masters-pool-postgres` (the Cluster instance, dependsOn cnpg-operator).

## Regenerate / upgrade

Pick the target chart version from `helm search repo cnpg/cloudnative-pg --versions`
(chart `0.28.3` ships operator `1.29.1`), then:

```sh
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update cnpg

VER=1.29.1            # operator (appVersion) -> sub-dir name
CHART=0.28.3          # chart version

helm template cnpg cnpg/cloudnative-pg \
  --version "$CHART" \
  --namespace cnpg-system \
  --include-crds > /tmp/cnpg-all.yaml

# Split: CustomResourceDefinition docs -> $VER/crds/cnpg-crds.yaml, the rest -> $VER/operator/cnpg-operator.yaml
```

(The split is a simple "route CustomResourceDefinition docs to crds/, everything else to
operator/" pass — see the kustomization.yaml files for the expected file names.)

After committing a new `$VER/` dir, bump the `path:` of the `cnpg-crds` / `cnpg-operator`
Flux Kustomizations to the new version and delete the old dir once rolled out.

## Backups

Backups use the **Barman Cloud Plugin** (CNPG-I), the modern method for CNPG 1.26+ — see
`../barman-cloud-plugin/`. Config lives in an `ObjectStore` CR and is referenced from
`Cluster.spec.plugins`, not the deprecated in-tree `.spec.backup.barmanObjectStore`.
