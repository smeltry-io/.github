# Smeltry

**Self-service bare metal infrastructure portal.**

Smeltry transforms physical servers into usable infrastructure — Kubernetes clusters,
bare OS servers, and more — through a Kubernetes-native, self-service portal.

> *The bare metal is the raw ore. Smeltry is the process that shapes it.*

---

## What It Does

Users provision infrastructure through a web UI (Headlamp plugin) or CLI (`smeltry`).
Under the hood, Smeltry orchestrates a battle-tested open-source stack:

| Layer | Tool |
|---|---|
| Identity | [Authentik](https://goauthentik.io) (OIDC) |
| CMDB / IPAM | [Netbox](https://netbox.dev) |
| OS Provisioning | [Tinkerbell](https://tinkerbell.org) |
| Cluster Lifecycle | [Cluster API](https://cluster-api.sigs.k8s.io) + [CAPT](https://github.com/tinkerbell/cluster-api-provider-tinkerbell) |
| Hosted Control Planes | [Kamaji](https://kamaji.clastix.io) |
| Networking | [Cilium](https://cilium.io) (L2 announcements) |
| Addon Delivery | [Sveltos](https://projectsveltos.github.io/sveltos/) |

---

## Repositories

| Repo | Description |
|---|---|
| [smeltry-operator](https://github.com/smeltry-io/smeltry-operator) | Kubernetes operators: ClusterClaim, ServerClaim, NetboxTenant |
| [smeltry](https://github.com/smeltry-io/smeltry) | CLI — device flow OIDC, cluster and server management |
| [smeltry-headlamp](https://github.com/smeltry-io/smeltry-headlamp) | Headlamp plugin — self-service web UI |
| [helm-charts](https://github.com/smeltry-io/helm-charts) | Helm charts for deploying Smeltry itself |
| [machinecfg](https://github.com/smeltry-io/machinecfg) | Netbox → Tinkerbell Hardware sync |
| [.github](https://github.com/smeltry-io/.github) | Org-wide governance, templates, and policies |

---

## Status

**Early stage — not yet v1.** The project is under active development.
Star the repo or watch releases to follow progress.

---

## License

Apache License 2.0 — see [LICENSE](https://github.com/smeltry-io/.github/blob/main/LICENSE).
