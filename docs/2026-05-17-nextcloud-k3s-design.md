# Nextcloud on k3s — Design Spec

Date: 2026-05-17
Status: Approved; implementation plan written
(`docs/superpowers/plans/2026-05-17-nextcloud-k3s-deployment.md`).
Amended 2026-05-17 for remote-sudo automation (see §5).
Cluster ref: `/home/dustfeather/cloudflare-mesh-k3s-state.md`

## 1. Goal & scope

Host a **family / small-team groupware** Nextcloud (files + calendar/contacts/
notes/photos; several users; 100s of GB over time) on the existing 3-node
k3s-over-Cloudflare-WARP-Mesh homelab, accessible from laptops **and an
Android phone**, with **no public exposure**.

In scope: Nextcloud app, database, cache, storage, Mesh-only access, TLS,
backups, node placement.
Out of scope (deferred, addable later, no infra blocker): Collabora/OnlyOffice
online editing; Photos/Memories app (it's just a Nextcloud app, no infra).

## 2. Constraints (from the cluster)

- Storage = `local-path` only (node-local, RWO, PV node-affinity). Chosen
  data backend: **local-path on `asus-laptop`** (421 GB free NVMe). No
  storage HA: if asus is down, Nextcloud is down. Accepted by user.
- Inter-node link = Cloudflare WARP Mesh only; intermittent/small acer & wsl.
  ⇒ entire stack **pinned to `asus-laptop`** (like `degoog`).
- No ingress controller (traefik disabled). Service exposure = NodePort on
  the asus Mesh IP `100.96.0.2`, consistent with headlamp (`:30443`).
- cloudflared tunnel is token-managed; **not used here** (Mesh-only choice).

## 3. Architecture

Namespace `nextcloud`; all pods `nodeSelector: kubernetes.io/hostname=asus-laptop`.

| Component | Choice | Notes |
|---|---|---|
| App | official `nextcloud/nextcloud` Helm chart (repo `https://nextcloud.github.io/helm/`) | nginx sidecar terminates TLS |
| Database | **MariaDB**, dedicated, local-path PVC | chart-managed subchart preferred; standalone Deployment acceptable fallback. (Postgres explicitly not chosen.) |
| Cache / file-locking | **Valkey** (Redis-compatible), dedicated Deployment+Service | Nextcloud `memcache.locking` + `memcache.distributed` → this Valkey. Required for multi-user correctness. Not reusing `degoog/valkey`. |
| TLS | **cert-manager** (new cluster component, ns `cert-manager`) + ClusterIssuer **Cloudflare DNS-01** | valid Let's Encrypt cert for `nextcloud.itguys.ro`; **auto-renewed** (~30d pre-expiry, no manual action) |

### Storage (local-path on asus)
- `nextcloud-data` — **300Gi** (bulk user files).
- `nextcloud-db` — ~20Gi (MariaDB).
- `nextcloud-backups` — ~100Gi (local quick-restore copy; NOT the durability
  guarantee — see Backups).

### Access (Mesh-only)
- DNS: `nextcloud.itguys.ro` → A `100.96.0.2`, **DNS-only / grey-cloud**
  (same pattern as `k3s.itguys.ro`). Resolves+reachable **only for Cloudflare
  WARP Mesh participants**.
- Exposure: the asus-pinned nginx TLS proxy binds the asus host's **`:443`
  via `hostPort`** (single replica), so access is `https://nextcloud.itguys.ro`
  with no port. (Superseded the initial NodePort `:30444` proposal — see the
  2026-05-18 amendment in §5.)
- Nextcloud config: `trusted_domains` includes `nextcloud.itguys.ro`;
  `overwrite.cli.url=https://nextcloud.itguys.ro`; `overwriteprotocol=https`.
- **Companion manual step (user/device side, not cluster):** install
  Cloudflare WARP / "Cloudflare One Agent" on Android, enroll in the
  `itguys` Zero Trust team, keep connected (ideally always-on). The phone
  uses the **Default** device profile (the `mesh-connectors` Include profile
  matches only connector pseudo-emails), which full-tunnels by default so
  `100.96.0.0/12` is reachable. Optional later: tune the phone profile's
  split tunnel if full-tunnel on the phone is undesirable.

### Data flow
Mesh client (laptop / WARP-enrolled phone) → `https://nextcloud.itguys.ro`
(A→100.96.0.2, WARP-encrypted transport) → asus host `:443` (proxy `hostPort`) →
nginx (TLS termination, cert-manager cert) → Nextcloud → MariaDB + Valkey
(in-cluster, same node) ; files on local-path PVC on asus.

### Security model
- Network boundary = the WARP Mesh (only enrolled team devices can route to
  `100.96.0.2`). No public exposure, no CF Access.
- Application auth = Nextcloud's own accounts; enforce 2FA for users; keep
  brute-force protection on.
- Transport = WARP/WireGuard encryption end-to-end **plus** HTTPS (the latter
  is for Nextcloud client/app compatibility — the mobile apps require
  `https://` and reject self-signed; not needed for wire secrecy, which WARP
  already provides).

## 4. Backups (the only on-disk copy is on one asus disk)

Nightly CronJob (pinned to asus):
1. `occ` consistent DB dump (`mysqldump`) + `config.php` → `nextcloud-backups` PVC.
2. `rsync` of `nextcloud-data` + the DB dump to `acer-laptop` over Mesh SSH
   (the real offsite-ish recovery copy; `nextcloud-backups` on the same disk
   only covers accidental deletion, not disk loss).
Retention: keep N daily (define in plan). **Replication ≠ backup** — this is
the recovery path; there is intentionally no live storage replication.

## 5. Prerequisites (implementation plan must cover)

1. Install **cert-manager** (Helm) into `cert-manager` ns.
2. Create a **scoped Cloudflare API token** (`Zone:DNS:Edit` +
   `Zone:Zone:Read` for `itguys.ro`), store as a Secret for the DNS-01
   ClusterIssuer. Long-lived; never manually rotated for cert renewal.
3. Create DNS `nextcloud.itguys.ro` A → `100.96.0.2` (DNS-only).
4. User enrolls the Android phone in the WARP Mesh (companion step above).

> **Amendment 2026-05-17 (remote sudo):** The operator has granted passwordless
> remote `sudo` over Mesh SSH on **both** `asus-laptop` (`100.96.0.2`) and
> `acer-laptop` (`100.96.0.4`). Consequence for this design: host-side prep for
> the off-node backup target (create `~/backup/nextcloud/`, authorize the
> dedicated backup SSH key on acer — §4 step 2) is **no longer a manual user
> step**; the implementation plan automates it. The only items that remain
> genuinely human/out-of-band are the ones that are *not* laptop shell actions:
> the Cloudflare dashboard API token (#2), the DNS record (#3), and the Android
> WARP enrolment (#4). The single-node pin (§2/§3) and versioned-imperative
> apply model (footer) are deliberate choices, unaffected by host access.

> **Amendment 2026-05-18 (default :443 instead of NodePort :30444):** On user
> request, the nginx TLS front-proxy now binds the **asus host `:443` directly
> via `hostPort`** (the pod is single-replica and asus-pinned; nginx master
> runs as root so it may bind the privileged port; k3s NodePorts cannot use
> 443 as they are restricted to 30000–32767). The proxy `Service` is therefore
> `ClusterIP` (in-cluster only); external reach is the hostPort on
> `100.96.0.2:443`. Access URL is now `https://nextcloud.itguys.ro` (no port);
> `overwritehost`/`overwrite.cli.url` are portless accordingly. Proxy
> Deployment strategy is `Recreate` (only one pod may own host :443 at a time).
> This keeps a single shared :443 TLS entrypoint: **future apps** are added as
> additional nginx `server {}` SNI blocks + their own cert-manager
> `Certificate` (it can later graduate to a real ingress controller without
> re-deciding this). No k3s reconfiguration or control-plane restart was
> needed. Mesh-only, no public exposure, and all §7 acceptance criteria are
> unchanged (they never specified a port).
>
> **Gateway TLS-inspection caveat (discovered during the :443 cutover):** the
> `itguys` Zero Trust org has Cloudflare **Gateway `tls_decrypt` enabled
> org-wide**. It inspects standard `:443` but *not* arbitrary high ports — so
> the old NodePort `:30444` accidentally evaded it, while `:443` was being
> TLS-MITM'd (clients saw the *Gateway CA*, not Let's Encrypt — breaks the
> Nextcloud Android app and §7 "valid cert, no warnings", and means Cloudflare
> decrypts private file traffic, contrary to the security model). **Required
> out-of-band Cloudflare dependency:** a Gateway HTTP **Do Not Inspect** policy
> (`action: off`, selector **`http.conn.hostname == "nextcloud.itguys.ro"`** —
> the SNI field; the decrypted `http.request.host` is NOT valid for the `off`
> action) so the real LE cert flows end-to-end on `:443`. Created via the
> account-scoped CF API token (rule id `df440536-0b50-483d-b5d7-70cd7cbe6230`,
> precedence 50). Any *future* app put on `:443` behind this proxy needs its
> hostname added to (or a companion of) this Do-Not-Inspect rule.

## 6. Risks / accepted trade-offs

- **No storage HA**: asus down ⇒ Nextcloud down. Accepted (local-path choice);
  recovery = backups + restore.
- **300Gi reserved on asus** (421Gi free) — monitor growth; revisit object
  storage (R2) if it approaches disk limits (was offered, declined for now).
- **Phone needs WARP connected** for sync; Android Doze/battery may interfere
  with always-on VPN. Accepted (Mesh-only choice).
- cert-manager + Cloudflare API token are new trust dependencies (scoped,
  durable).

## 7. Acceptance criteria

- `https://nextcloud.itguys.ro` reachable from a Mesh laptop **and** the
  WARP-enrolled phone; **valid cert, no browser/app warnings**.
- Web login works; **Nextcloud Android app** logs in and syncs files +
  CalDAV/CardDAV.
- MariaDB + Valkey healthy; Nextcloud security/setup checks clean
  (HTTPS, locking via Redis, trusted domains).
- Nightly backup produces a restorable DB dump + data copy on asus **and**
  an `acer-laptop` copy.
- cert-manager `Certificate` is `Ready=True` with a future `renewalTime`
  (auto-renew confirmed; no manual step).
- All Nextcloud-stack pods on `asus-laptop`; nothing scheduled to acer/wsl.

## 8. Deferred (YAGNI)

Collabora/OnlyOffice (heavy, separate component); Photos/Memories app
(no infra impact). Add later without redesign.

---
Repo: versioned in `/home/dustfeather/projects/nextcloud` (git; lightweight
**versioned-imperative** — manual `helm`/`kubectl apply`, no GitOps
controller). **Secrets are never committed** — template + gitignore pattern,
see repo `README.md` / `secrets/`. Cluster operational source-of-truth
remains `/home/dustfeather/cloudflare-mesh-k3s-state.md`.
