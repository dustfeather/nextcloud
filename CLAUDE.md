# CLAUDE.md

Deployment artifacts for Nextcloud on existing 3-node k3s-over-Cloudflare-WARP-Mesh homelab. Not app code — no build/lint/test. Live at `https://nextcloud.itguys.ro` (Mesh-only, :443 via proxy hostPort).

## Operating model: versioned-imperative (no GitOps)

- No GitOps controller; nothing here auto-applied. Changes reach cluster only via manual `helm upgrade -f ...` / `kubectl apply -f ...` by operator.
- Adding artifact → also update apply-order docs in `README.md`. Repo = record; human = deployer.
- No cluster CI — validate before propose-apply: `helm template <release> <chart> -f helm/<values>.yaml` + `helm lint`; `kubectl apply --dry-run=server -f manifests/<file>.yaml`.

## Source-of-truth (read before cluster-affecting work)

- `docs/2026-05-17-nextcloud-k3s-design.md` — approved design. Decisions settled unless user reopens.
- `~/cloudflare-mesh-k3s-state.md` (outside repo) — cluster runbook; SoT for node names, Mesh IPs, existing components (degoog, headlamp). This repo intentionally doesn't duplicate.

## Architecture invariants (don't violate without revisiting design)

- **Single-node pin:** whole stack in ns `nextcloud` w/ `nodeSelector: kubernetes.io/hostname=asus-laptop` on every pod. Storage = `local-path` (node-local RWO) on asus only. Nothing may schedule to `acer-laptop` or `wsl`. No storage HA — accepted trade-off.
- **Mesh-only, no public:** asus-pinned nginx TLS proxy binds asus host `:443` via `hostPort` (single replica; Service ClusterIP). DNS `nextcloud.itguys.ro` → A `100.96.0.2`, DNS-only / grey-cloud. Cluster cloudflared tunnel deliberately NOT used. No ingress controller (traefik disabled) — this proxy = single shared :443 TLS entrypoint; future apps = added nginx SNI server blocks + own cert.
- **Components:** official `nextcloud/nextcloud` Helm chart (Apache image, plain HTTP on Service :8080; chart's nginx sidecar disabled — TLS terminated by separate front nginx, plan Deviation #1). Dedicated MariaDB (Postgres explicitly rejected). Dedicated Valkey for `memcache.locking`/`memcache.distributed` — fresh instance, do NOT reuse degoog's Valkey. TLS via cert-manager (ns `cert-manager`) w/ Cloudflare DNS-01 ClusterIssuer, auto-renewed (proxy self-reloads on rotation).
- **Backups ≠ replication:** nightly asus-pinned CronJob does `mariadb-dump --single-transaction` (no `occ`/maintenance-mode — plan Deviation #2) + `config.php` copy to local PVC, then `rsync` data + dump to `acer-laptop` over Mesh SSH. Acer copy = real recovery path; local PVC only covers accidental deletion. No live storage replication by design.

## Secrets policy (hard rule — repo may go to GitHub)

- Never commit plaintext. `.gitignore` enforces — do NOT loosen.
- Pattern: commit `secrets/<name>.example` (placeholders) → operator copies to `secrets/<name>.yaml` (gitignored, real) → `kubectl apply` out-of-band.
- Five secrets: scoped Cloudflare API token (`Zone:DNS:Edit` + `Zone:Zone:Read` on `itguys.ro`; `cf-api-token`), Nextcloud admin (`nextcloud-admin`), MariaDB root + nextcloud DB (`nextcloud-db`), Valkey (`valkey-auth`), backup SSH key (`backup-ssh`).
- Non-k8s dependency: out-of-band Cloudflare Gateway "Do Not Inspect" rule for `nextcloud.itguys.ro` required (see README "Operational dependencies" / design §5).
