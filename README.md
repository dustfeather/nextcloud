# nextcloud (k3s homelab)

Lightweight **versioned-imperative** repo for the Nextcloud deployment on the
3-node k3s-over-Cloudflare-WARP-Mesh cluster. No GitOps controller — changes
are applied manually (`helm upgrade -f ...`, `kubectl apply -f ...`) but every
declarative artifact is version-controlled here.

Cluster operational runbook / source-of-truth: `~/cloudflare-mesh-k3s-state.md`.

## Layout

```
docs/        design spec(s)
helm/        Helm values (committed, secret-free): cert-manager + nextcloud
manifests/   raw k8s YAML: namespace, cert-manager issuer + certificate,
             MariaDB, Valkey, PVCs, nginx TLS proxy (hostPort :443), backup CronJob
secrets/     templates ONLY (*.example). Real secrets are gitignored and
             applied out-of-band.
```

## Secrets policy (read before committing anything)

**Never commit plaintext secrets.** This repo may end up on GitHub.
Excluded by `.gitignore`: everything in `secrets/` except `*.example`, plus
`*-secret.yaml`, `*.key`, `*.pem`, `kubeconfig*`, `.env*`, `*-token*`.

Workflow:
1. A template lives at `secrets/<thing>.example` (placeholders, committed).
2. Copy → `secrets/<thing>.yaml` (real values, **gitignored**).
3. `kubectl apply -f secrets/<thing>.yaml` out-of-band.

Secrets this deployment needs (all out-of-band, never in git):
- Cloudflare API token for cert-manager DNS-01 (`Zone:DNS:Edit` +
  `Zone:Zone:Read` on `itguys.ro`) — `secrets/cf-api-token`.
- Nextcloud admin password — `secrets/nextcloud-admin`.
- MariaDB root + nextcloud DB passwords — `secrets/nextcloud-db`.
- Valkey password — `secrets/valkey-auth`.
- Dedicated backup SSH private key (rsync to acer) — `secrets/backup-ssh`.

(If this grows, consider SOPS+age or Sealed Secrets — out of scope for now.)

## Apply order

Plan: `docs/superpowers/plans/2026-05-17-nextcloud-k3s-deployment.md`.
Human prereqs P1–P4 are in that plan's "Prerequisites" section.

1. `kubectl apply -f manifests/00-namespace.yaml`
2. `helm install cert-manager jetstack/cert-manager -n cert-manager --create-namespace --version v1.20.2 -f helm/cert-manager-values.yaml`
3. `kubectl apply -f secrets/cf-api-token.yaml` (out-of-band) → `kubectl apply -f manifests/10-clusterissuer-letsencrypt.yaml`
4. `kubectl apply -f manifests/20-certificate.yaml`
5. `kubectl apply -f secrets/nextcloud-db.yaml` (oob) → `kubectl apply -f manifests/30-mariadb.yaml`
6. `kubectl apply -f secrets/valkey-auth.yaml` (oob) → `kubectl apply -f manifests/40-valkey.yaml`
7. `kubectl apply -f manifests/50-nextcloud-pvcs.yaml`
8. `kubectl apply -f secrets/nextcloud-admin.yaml` (oob) → `helm install nextcloud nextcloud/nextcloud -n nextcloud --version 9.1.0 -f helm/nextcloud-values.yaml`
9. `kubectl apply -f manifests/60-nginx-tls-proxy.yaml` (proxy binds asus host :443 via hostPort)
10. Create DNS `nextcloud.itguys.ro` A → 100.96.0.2 (DNS-only); verify end-to-end.
11. `kubectl apply -f secrets/backup-ssh.yaml` (oob) → `kubectl apply -f manifests/70-backup-cronjob.yaml`

Access: `https://nextcloud.itguys.ro` (default :443, Mesh participants only).
Upgrades: `helm upgrade nextcloud nextcloud/nextcloud -n nextcloud --version <v> -f helm/nextcloud-values.yaml`.

## Status

Deployed. Plan executed: docs/superpowers/plans/2026-05-17-nextcloud-k3s-deployment.md. Access https://nextcloud.itguys.ro (Mesh only).

## Operational dependencies (silent-failure if removed — read before touching Cloudflare/PVs)

- **Cloudflare Gateway "Do Not Inspect" rule** — the `itguys` org has Gateway
  `tls_decrypt` enabled, which TLS-MITMs `:443`. Rule id
  `df440536-0b50-483d-b5d7-70cd7cbe6230` (`action: off`,
  `http.conn.hostname == "nextcloud.itguys.ro"`) exempts this host so the real
  Let's Encrypt cert is served end-to-end. **If this rule is deleted/disabled
  the failure is silent**: clients get the Cloudflare Gateway CA, the Nextcloud
  Android app breaks, and file traffic is decrypted at Cloudflare's edge.
  Verify: `echo | openssl s_client -connect 100.96.0.2:443 -servername
  nextcloud.itguys.ro 2>/dev/null | openssl x509 -noout -issuer` must show
  `O = Let's Encrypt` (NOT `Gateway CA`). Any future app added on :443 needs
  its hostname added to a Do-Not-Inspect rule. Full context: design doc
  §5 amendment 2026-05-18 + `~/cloudflare-mesh-k3s-state.md`.
- **PV reclaim policy = Retain** — the bound PVs for all three claims
  (`nextcloud-data`, `nextcloud-db`, `nextcloud-backups`) were patched to
  `persistentVolumeReclaimPolicy: Retain` (local-path defaults to `Delete`), so
  an accidental `kubectl delete pvc` does not wipe the hostPath. Disk-loss
  recovery is still the nightly acer-laptop rsync (design §4); these PVs are
  single-disk on asus.
