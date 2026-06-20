# Barman Cloud Plugin (CNPG-I) — vendored Helm output

The **modern** CloudNativePG backup method (CNPG 1.26+): a CNPG-I plugin instead of the
deprecated in-tree `.spec.backup.barmanObjectStore`. Backups are configured with an
`ObjectStore` CR and referenced from `Cluster.spec.plugins`.

Rendered (`helm template`) and committed, versioned by plugin release (`v0.13.0/`), with the
CRD split from the workload so Flux applies them in order.

```
v0.13.0/                # plugin (appVersion) version
  crds/                 #   ObjectStore CRD          -> Flux Kustomization: barman-plugin-crds
  plugin/               #   Deployment/Svc/RBAC/certs -> Flux Kustomization: barman-plugin
                        #     (dependsOn: barman-plugin-crds, cnpg-operator, platform-controllers)
```

Requires **cert-manager** (operator↔plugin mTLS) and the **CNPG operator** (the plugin
registers via its Service in `cnpg-system`). Plugin name used in `Cluster.spec.plugins`:
`barman-cloud.cloudnative-pg.io`.

## Regenerate / upgrade

```sh
helm repo add cnpg https://cloudnative-pg.github.io/charts && helm repo update cnpg
# chart 0.7.0 ships plugin v0.13.0  (helm search repo cnpg/plugin-barman-cloud --versions)
helm template barman-cloud cnpg/plugin-barman-cloud \
  --version 0.7.0 --namespace cnpg-system --include-crds > /tmp/barman-all.yaml
# Split: CustomResourceDefinition -> v0.13.0/crds/objectstore-crd.yaml, rest -> v0.13.0/plugin/barman-cloud-plugin.yaml
```
