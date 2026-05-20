# CLAUDE.md

Guidance for Claude Code working in this repository.

## Nature of this repo

- Deployment artifacts for Nextcloud on an existing 3-node k3s-over-Cloudflare-WARP-Mesh homelab. Not an application codebase: there is no build/lint/test.
- State: deployed and live at `https://nextcloud.itguys.ro` (Mesh-only, :443 via proxy hostPort). `helm/`, `manifests/`, `secrets/*.example` are populated.

## Operating model: versioned-imperative (no GitOps)

- No GitOps controller; nothing here is auto-applied. Changes reach the cluster only via manual `helm upgrade -f ...` / `kubectl apply -f ...` run out-of-band by the operator.
- When adding an artifact, also update the apply-order docs in `README.md`. The repo is the record; the human is the deployer.
- No cluster CI — validate before proposing an apply: `helm template <release> <chart> -f helm/<values>.yaml` + `helm lint`; `kubectl apply --dry-run=server -f manifests/<file>.yaml`.

## Source-of-truth files (read before any cluster-affecting work)

- `docs/2026-05-17-nextcloud-k3s-design.md` — approved design spec. Treat its decisions as settled unless the user reopens them.
- `~/cloudflare-mesh-k3s-state.md` (outside this repo, in user's home) — cluster runbook; source of truth for node names, Mesh IPs, existing components (`degoog`, headlamp). This repo intentionally does not duplicate it.

## Architecture invariants (don't violate without revisiting the design)

- Single-node pin: whole stack in namespace `nextcloud` with `nodeSelector: kubernetes.io/hostname=asus-laptop` on every pod. Storage is `local-path` (node-local RWO) on asus only. Nothing may schedule to `acer-laptop` or `wsl`. No storage HA — accepted trade-off.
- Mesh-only, no public exposure: asus-pinned nginx TLS proxy binds asus host `:443` via `hostPort` (single replica; its Service is ClusterIP). DNS `nextcloud.itguys.ro` → A `100.96.0.2`, DNS-only / grey-cloud. The cluster cloudflared tunnel is deliberately not used. No ingress controller (traefik disabled) — this proxy is the single shared :443 TLS entrypoint; future apps = added nginx SNI server blocks + own cert.
- Components: official `nextcloud/nextcloud` Helm chart (Apache image, plain HTTP on Service :8080; chart's nginx sidecar disabled — TLS terminated by the separate front nginx proxy, plan Deviation #1). Dedicated MariaDB (Postgres explicitly rejected). Dedicated Valkey for `memcache.locking`/`memcache.distributed` — a fresh instance, do not reuse `degoog`'s Valkey. TLS via cert-manager (ns `cert-manager`) with a Cloudflare DNS-01 ClusterIssuer, auto-renewed (proxy self-reloads on rotation).
- Backups ≠ replication: nightly asus-pinned CronJob does `mariadb-dump --single-transaction` (no `occ`/maintenance-mode — plan Deviation #2) + a `config.php` copy to a local PVC, then `rsync` data + dump to `acer-laptop` over Mesh SSH. The acer copy is the real recovery path; the local PVC only covers accidental deletion. No live storage replication by design.

## Secrets policy (hard rule — this repo may go to GitHub)

- Never commit plaintext secrets. `.gitignore` enforces this — do not loosen it.
- Pattern: commit `secrets/<name>.example` (placeholders) → operator copies to `secrets/<name>.yaml` (gitignored, real values) → `kubectl apply` out-of-band.
- Five secrets: scoped Cloudflare API token (`Zone:DNS:Edit` + `Zone:Zone:Read` on `itguys.ro`; `cf-api-token`), Nextcloud admin password (`nextcloud-admin`), MariaDB root + nextcloud DB passwords (`nextcloud-db`), Valkey password (`valkey-auth`), backup SSH key (`backup-ssh`).
- Non-k8s dependency: an out-of-band Cloudflare Gateway "Do Not Inspect" rule for `nextcloud.itguys.ro` is required (see README "Operational dependencies" / design §5).
