# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Deployment artifacts for **Nextcloud on an existing 3-node k3s-over-Cloudflare-WARP-Mesh
homelab**. It is **not** an application codebase — there is no build/lint/test. It holds
version-controlled declarative artifacts (Helm values, raw k8s YAML, secret templates)
plus the design spec.

**Current state: design-only.** As of the latest commit, `helm/`, `manifests/`, and
`secrets/*.example` are empty placeholders — only their `README.md`s exist. The design
is approved; the implementation plan has not been written and **no cluster changes have
been made**. The next workflow step is writing the implementation plan (use the
`superpowers:writing-plans` skill), which is what populates `helm/`, `manifests/`,
`secrets/*.example`, and the "Apply order" in the root `README.md`.

## Operating model: versioned-imperative (no GitOps)

There is **no GitOps controller**. Nothing in this repo is auto-applied. Every
declarative artifact is committed here, but changes reach the cluster only through
**manual** `helm upgrade -f ...` / `kubectl apply -f ...` run out-of-band by the
operator. When adding artifacts, also update the apply-order documentation in
`README.md` — the repo is the record, the human is the deployer.

Validate before proposing an apply (no cluster CI exists):
- `helm template <release> <chart> -f helm/<values>.yaml` and `helm lint`
- `kubectl apply --dry-run=server -f manifests/<file>.yaml`

## Source-of-truth files (read before any cluster-affecting work)

- `docs/2026-05-17-nextcloud-k3s-design.md` — the full approved design spec
  (architecture, storage sizes, access model, backup strategy, acceptance criteria).
  Treat its decisions as settled unless the user reopens them.
- `~/cloudflare-mesh-k3s-state.md` (**outside this repo**, in the user's home) — the
  cluster operational runbook and source-of-truth for node names, Mesh IPs, existing
  components (e.g. `degoog`, headlamp). Consult it for live cluster facts; this repo
  intentionally does not duplicate it.

## Architecture invariants (don't violate without revisiting the design)

- **Single-node pin.** The entire stack runs in namespace `nextcloud` with
  `nodeSelector: kubernetes.io/hostname=asus-laptop` on every pod. Storage is
  `local-path` (node-local RWO) on asus only. Nothing may schedule to `acer-laptop`
  or `wsl`. No storage HA is by design (accepted trade-off).
- **Mesh-only, no public exposure.** The asus-pinned nginx TLS proxy binds the
  asus host `:443` via `hostPort` (single replica; its Service is ClusterIP),
  so access is `https://nextcloud.itguys.ro` (default port, no NodePort), with
  DNS `nextcloud.itguys.ro` → A `100.96.0.2` **DNS-only / grey-cloud**. The
  cluster's cloudflared tunnel is deliberately **not** used here. There is no
  ingress controller (traefik disabled); this proxy is the single shared :443
  TLS entrypoint (future apps = added nginx SNI server blocks + their own cert).
- **Components:** official `nextcloud/nextcloud` Helm chart (nginx sidecar terminates
  TLS) + dedicated **MariaDB** (Postgres explicitly rejected) + dedicated **Valkey**
  for `memcache.locking`/`memcache.distributed` (a fresh instance — do not reuse
  `degoog`'s Valkey). TLS via **cert-manager** (ns `cert-manager`) with a Cloudflare
  **DNS-01** ClusterIssuer, auto-renewed.
- **Backups ≠ replication.** Nightly asus-pinned CronJob: `occ`/`mysqldump` +
  `config.php` to a local PVC, then `rsync` data + dump to `acer-laptop` over Mesh
  SSH. The acer copy is the real recovery path; the local PVC only covers accidental
  deletion. There is intentionally no live storage replication.

## Secrets policy (hard rule — this repo may go to GitHub)

**Never commit plaintext secrets.** `.gitignore` permits only `README.md`,
`*.example`, and `*.template` under `secrets/`, and globally excludes `*-secret.yaml`,
`*.key`, `*.pem`, `kubeconfig*`, `.env*`, `*-token*`. Pattern: commit
`secrets/<name>.example` (placeholders) → operator copies to `secrets/<name>.yaml`
(gitignored, real values) → `kubectl apply` out-of-band. Secrets needed: scoped
Cloudflare API token (`Zone:DNS:Edit` + `Zone:Zone:Read` on `itguys.ro`), Nextcloud
admin password, MariaDB root/user passwords.
