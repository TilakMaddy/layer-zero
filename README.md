# layer-zero

A Flux platform package: secrets, an Envoy Gateway underlay with cert-manager and
CNPG, and a Grafana observability stack. A consumer points one Kustomization at
`platform/bootstrap` and the package declares everything else.

## Consuming it

Vendor the repo (submodule or subtree), then declare four things in
`flux-system`: a `GitRepository` that can read it, two `ConfigMap`s named
`platform-vars` and `platform-owner-vars`, and one `Kustomization`.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: bootstrap
  namespace: flux-system
spec:
  path: ./path/where/you/vendored/platform/bootstrap
  prune: true
  wait: true
  sourceRef:
    kind: GitRepository
    name: platform-foundation
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: platform-vars
      - kind: ConfigMap
        name: platform-owner-vars
```

That is the whole call. There is no `substitute:` block anywhere in this package —
those two ConfigMaps are the entire interface, and every Kustomization the package
creates reads both by those names.

`wait: true` matters: the package's stages are ordered, and anything you layer on
top should depend on this Kustomization rather than on a stage inside it.

### The contract

Both names are fixed — the package reads `ConfigMap/platform-vars` and
`ConfigMap/platform-owner-vars` in `flux-system` directly. Every key below is
required.

`platform-owner-vars` holds what belongs to whoever runs the cluster, so a new
operator has one small file to edit:

| key | |
|---|---|
| `CLUSTER_ZONE` | DNS zone the gateway serves, e.g. `example.com` |
| `TXT_OWNER_ID` | external-dns `--txt-owner-id`; must be unique among clusters sharing a DNS zone, and changing it on a live cluster orphans the TXT records the old id owns |
| `OP_VAULT` | 1Password vault name |

`platform-vars` holds the rest:

| key | |
|---|---|
| `PLATFORM_SOURCE` | name of your `GitRepository` pointing at this repo |
| `PLATFORM_PATH_PREFIX` | where you vendored it, relative to the source root; `.` when the package is the source itself |
| `PLATFORM_NAMESPACE` | namespace to create and install the gateway into |
| `ACME_ENV` | `staging` or `production` |
| `ENVOY_REPLICAS`, `SHIM_REPLICAS` | quoted integers |
| `FLUX_INTERVAL`, `FLUX_RETRY_INTERVAL`, `FLUX_TIMEOUT` | applied to every Kustomization the package creates |
| `GRAFANA_ALLOWED_CIDRS` | client CIDRs allowed to reach the Grafana route, as a YAML flow sequence, e.g. `'["0.0.0.0/0"]'`; everything else is denied at the gateway |
| `OP_VAULT_CLOUDFLARE_TOKEN` | `<item>/[section/]<field>` — DNS-01 and external-dns |
| `OP_VAULT_GRAFANA_ADMIN_USER`, `OP_VAULT_GRAFANA_ADMIN_PASSWORD` | Grafana login |
| `OP_VAULT_RESEND_SMTP_PASSWORD` | SMTP password |
| `OP_VAULT_ACME_EMAIL` | Let's Encrypt account address |
| `OP_VAULT_ALERT_EMAIL_TO` | contact point address |
| `OP_VAULT_GRAFANA_SMTP_HOST`, `OP_VAULT_GRAFANA_SMTP_USER`, `OP_VAULT_GRAFANA_SMTP_FROM_ADDRESS` | alert delivery |

Every `OP_VAULT_*` value is an address inside the vault, written the way 1Password
writes it minus the `op://<vault>/` prefix.

### Prerequisite

Secrets come from 1Password through External Secrets. Seed the service account
token before Flux starts, since the package cannot fetch its own credential:

```sh
kubectl create namespace external-secrets-system
kubectl create secret generic onepassword-token \
  --namespace=external-secrets-system \
  --from-literal=token="$OP_SERVICE_ACCOUNT_TOKEN"
```

## What it creates

Three stages, in order, each waiting on the last.

- **`secrets`** — External Secrets Operator and Stakater Reloader, the
  `onepassword` `ClusterSecretStore`, and the Cloudflare token as a
  `ClusterExternalSecret`.
- **`underlay`** — cert-manager, CNPG, Kyverno, Keel, external-dns, Envoy Gateway;
  the `ClusterIssuer`, the CNPG barman-cloud backup plugin, the gateway and its
  certificate, and the Postgres STARTTLS shim.
- **`observability`** — kube-prometheus-stack, grafana-operator, Alloy, Loki, Tempo,
  and a Grafana instance with datasources and alerting.

Three fixed names: the package expects `ConfigMap/platform-vars` and
`ConfigMap/platform-owner-vars` from you, and produces `Secret/platform-secret-vars`
in `flux-system` — the Grafana credentials, the SMTP settings and the ACME address,
resolved from 1Password and read back by the package's own stages. None is
configurable, because a name the caller cannot meaningfully change is a contract,
not a parameter.

## Conventions

- One namespace per operator. The exception is `plugin-barman-cloud`, which CNPG
  discovers by Service in its own namespace and so has to install into
  `cnpg-system`.
- Workloads reading a credential from the environment carry
  `reloader.stakater.com/auto: "true"`, so a rotated Secret reaches the process.
- Namespaces opt in to gateway routing with the
  `oatlabs.oatmilk.work/gateway-access: "true"` label, and routes opt in to DNS with
  the `oatlabs.oatmilk.work/external-dns: "true"` annotation.
