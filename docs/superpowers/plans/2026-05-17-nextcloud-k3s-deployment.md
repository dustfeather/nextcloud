# Nextcloud on k3s Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy a Mesh-only, TLS-secured, single-node-pinned Nextcloud (files + groupware) on the existing 3-node k3s-over-Cloudflare-WARP-Mesh cluster, with nightly backups.

**Architecture:** All workloads live in namespace `nextcloud`, pinned to `asus-laptop` via `nodeSelector` (local-path storage is node-local). Standalone MariaDB + standalone Valkey back the official Nextcloud Helm chart (Apache image, plain HTTP internally). A dedicated nginx **front reverse-proxy** Deployment terminates TLS with a cert-manager (Cloudflare DNS-01) Let's Encrypt certificate and binds the asus host's `:443` directly via `hostPort` (so the URL is `https://nextcloud.itguys.ro`, no port). A nightly CronJob dumps the DB + config and rsyncs data off-node to `acer-laptop`.

**Tech Stack:** k3s v1.35.4, Helm v4, `nextcloud/nextcloud` chart `9.1.0` (app 33.0.3), `jetstack/cert-manager` `v1.20.2`, official `mariadb:11.4`, `valkey/valkey:8-alpine`, `nginx:1.27-alpine`, local-path PVCs.

---

## Verification-driven (not unit-test TDD)

This is infrastructure, not application code — there is no unit-test harness. Each task follows the same discipline adapted to k8s: **(1) write the verification command and confirm the resource is currently absent/not-ready ("the failing test"), (2) write the artifact with complete content, (3) apply it, (4) re-run the verification and confirm the expected ready state, (5) commit.** Treat the verification command as the test: it must fail before and pass after.

## Deviations from the design spec (read before executing)

The approved design (`docs/2026-05-17-nextcloud-k3s-design.md`) is followed except for two deliberate, documented refinements — both still satisfy every §7 acceptance criterion:

1. **TLS termination: dedicated front nginx reverse-proxy Deployment, not a chart "nginx sidecar."** The Nextcloud chart's nginx sidecar cannot mount the cert-manager TLS secret without overriding chart templates with brittle, version-fragile config (would force placeholders). A separate, fully-owned nginx Deployment in front of the chart's plain-HTTP ClusterIP Service mounts `nextcloud-tls` directly, terminates TLS, and binds the asus host's `:443` via `hostPort` (single replica, asus-pinned). Net effect on §7 is identical: valid LE cert, HTTPS only on the default port, no ingress controller, reachable only on the asus Mesh IP, all pods on asus. (Originally implemented as a NodePort `:30444`; cut over to `hostPort 443` on user request — see the 2026-05-18 amendment.)
2. **DB consistency: `mariadb-dump --single-transaction`, not an `occ maintenance:mode` toggle.** A single-transaction dump of InnoDB tables is a consistent snapshot and removes the need to exec `occ` / flip maintenance mode from the CronJob (fewer moving parts, no Nextcloud-container coupling). config.php is captured from the shared data PVC. §7's "restorable DB dump + data copy" is met.

## Prerequisites — human, out-of-band (must be done before Task 3 / Task 10)

These require the **Cloudflare dashboard or a physical phone** — they are *not* laptop shell actions, so remote sudo cannot automate them. The plan blocks on them where noted.

- **P1 — Cloudflare API token** (before Task 3): create a token scoped `Zone:DNS:Edit` + `Zone:Zone:Read` on zone `itguys.ro`. Long-lived; never rotated for cert renewal.
- **P2 — DNS A record** (before Task 10): `nextcloud.itguys.ro` → A `100.96.0.2`, **DNS-only / grey-cloud** (same as `k3s.itguys.ro`). Not needed for DNS-01 issuance (Task 4), only for end-to-end access.
- **P3 — Android WARP enrolment** (before final §7 acceptance): enrol the phone in the `itguys` Zero Trust team, keep WARP always-on (Default full-tunnel profile reaches `100.96.0.0/12`).

> **Former P4 (acer backup-target host prep) is now automated** in Task 11 Step 1 — passwordless remote sudo on `acer-laptop` is available (see Conventions). It is no longer a blocking human prerequisite.

> **Status (verified 2026-05-17): all satisfied.** **P1 ✓** token at `/home/dustfeather/.cf-token` (used by Task 3 Step 1). **P2 ✓** `nextcloud.itguys.ro` → `100.96.0.2`. **P3 ✓** phone enrolled + tested.

## Conventions

- Run all `kubectl`/`helm` from the environment where they already work (this WSL session; context `k3s-itguys`). If a stale default context resurfaces, add `--kube-context k3s-itguys`.
- **Remote host access:** passwordless `sudo` over Mesh SSH is available on both `asus-laptop` (`ssh asus-laptop` / `dustfeather@100.96.0.2`) and `acer-laptop` (`dustfeather@100.96.0.4`). Host-side setup is therefore automated in-plan, not deferred to the operator. SSH over the WARP Mesh can occasionally drop a *fresh* connect transiently — retry once before treating an SSH failure as real (an already-open session rides through the hiccup).
- Real secret files (`secrets/*.yaml`) are **gitignored** — never `git add` them. Only `secrets/*.example` are committed.
- Commit after every task. Branch: work on `main` is acceptable for this repo (versioned-imperative, single operator), but create `nextcloud-deploy` if you prefer isolation.

---

## Task 1: Namespace + secret templates

**Files:**
- Create: `manifests/00-namespace.yaml`
- Create: `secrets/cf-api-token.example`
- Create: `secrets/nextcloud-admin.example`
- Create: `secrets/nextcloud-db.example`
- Create: `secrets/valkey-auth.example`
- Create: `secrets/backup-ssh.example`

- [ ] **Step 1: Verify the namespace does not exist (the failing test)**

Run: `kubectl get ns nextcloud`
Expected: `Error from server (NotFound): namespaces "nextcloud" not found`

- [ ] **Step 2: Write `manifests/00-namespace.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nextcloud
  labels:
    app.kubernetes.io/part-of: nextcloud
```

- [ ] **Step 3: Apply and verify (the passing test)**

Run: `kubectl apply -f manifests/00-namespace.yaml && kubectl get ns nextcloud`
Expected: `namespace/nextcloud created` then `nextcloud   Active`

- [ ] **Step 4: Write `secrets/cf-api-token.example`**

```yaml
# Copy to secrets/cf-api-token.yaml (gitignored), replace REPLACE_ME, then:
#   kubectl apply -f secrets/cf-api-token.yaml
# Scope: Zone:DNS:Edit + Zone:Zone:Read on itguys.ro
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token
  namespace: cert-manager
type: Opaque
stringData:
  api-token: "REPLACE_ME_CLOUDFLARE_API_TOKEN"
```

- [ ] **Step 5: Write `secrets/nextcloud-admin.example`**

```yaml
# Copy to secrets/nextcloud-admin.yaml (gitignored), replace REPLACE_ME, then apply.
apiVersion: v1
kind: Secret
metadata:
  name: nextcloud-admin
  namespace: nextcloud
type: Opaque
stringData:
  nextcloud-username: "admin"
  nextcloud-password: "REPLACE_ME_STRONG_ADMIN_PASSWORD"
```

- [ ] **Step 6: Write `secrets/nextcloud-db.example`**

```yaml
# Copy to secrets/nextcloud-db.yaml (gitignored), replace REPLACE_ME values, then apply.
# Used by BOTH the standalone MariaDB Deployment and the Nextcloud chart (externalDatabase).
apiVersion: v1
kind: Secret
metadata:
  name: nextcloud-db
  namespace: nextcloud
type: Opaque
stringData:
  mariadb-root-password: "REPLACE_ME_STRONG_ROOT_PASSWORD"
  db-username: "nextcloud"
  db-password: "REPLACE_ME_STRONG_DB_PASSWORD"
```

- [ ] **Step 7: Write `secrets/valkey-auth.example`**

```yaml
# Copy to secrets/valkey-auth.yaml (gitignored), replace REPLACE_ME, then apply.
apiVersion: v1
kind: Secret
metadata:
  name: valkey-auth
  namespace: nextcloud
type: Opaque
stringData:
  redis-password: "REPLACE_ME_STRONG_VALKEY_PASSWORD"
```

- [ ] **Step 8: Write `secrets/backup-ssh.example`**

```yaml
# Generate a dedicated keypair (NOT your personal key):
#   ssh-keygen -t ed25519 -N '' -f /tmp/nc-backup -C nextcloud-backup
# Put the PRIVATE key below. The matching PUBLIC key is installed onto acer
# automatically in Task 11 Step 1 (remote sudo) — no manual authorized_keys edit.
# Copy to secrets/backup-ssh.yaml (gitignored), then apply.
apiVersion: v1
kind: Secret
metadata:
  name: backup-ssh
  namespace: nextcloud
type: Opaque
stringData:
  id_ed25519: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    REPLACE_ME_WITH_PRIVATE_KEY_CONTENTS
    -----END OPENSSH PRIVATE KEY-----
```

- [ ] **Step 9: Confirm secret templates are tracked but real files are ignored**

Run: `git status --porcelain secrets/ && git check-ignore secrets/cf-api-token.yaml || echo "yaml correctly NOT ignored yet (file absent)"`
Expected: the five `*.example` files show as untracked/added; `git check-ignore` prints `secrets/cf-api-token.yaml` (ignored).

- [ ] **Step 10: Commit**

```bash
git add manifests/00-namespace.yaml secrets/*.example
git commit -m "feat(nextcloud): namespace + secret templates"
```

---

## Task 2: Install cert-manager

**Files:**
- Create: `helm/cert-manager-values.yaml`

- [ ] **Step 1: Verify cert-manager is absent (the failing test)**

Run: `kubectl get ns cert-manager; kubectl get crd certificates.cert-manager.io`
Expected: both `NotFound`.

- [ ] **Step 2: Write `helm/cert-manager-values.yaml`**

```yaml
# cert-manager is cluster-wide; controllers may run on any node (not pinned).
crds:
  enabled: true
replicaCount: 1
resources:
  requests:
    cpu: 10m
    memory: 64Mi
webhook:
  resources:
    requests:
      cpu: 10m
      memory: 32Mi
cainjector:
  resources:
    requests:
      cpu: 10m
      memory: 64Mi
```

- [ ] **Step 3: Install the chart**

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --version v1.20.2 \
  -f helm/cert-manager-values.yaml
```
Expected: `STATUS: deployed`.

- [ ] **Step 4: Verify all cert-manager pods are Ready (the passing test)**

Run: `kubectl -n cert-manager rollout status deploy/cert-manager --timeout=180s && kubectl -n cert-manager rollout status deploy/cert-manager-webhook --timeout=180s && kubectl -n cert-manager get pods`
Expected: three deployments (`cert-manager`, `cert-manager-webhook`, `cert-manager-cainjector`) all `1/1 Running`.

- [ ] **Step 5: Commit**

```bash
git add helm/cert-manager-values.yaml
git commit -m "feat(nextcloud): install cert-manager v1.20.2"
```

---

## Task 3: Cloudflare API token secret + ClusterIssuer

**Prerequisite:** P1 done (Cloudflare token created).

**Files:**
- Create: `manifests/10-clusterissuer-letsencrypt.yaml`

- [ ] **Step 1: Apply the real Cloudflare token secret (out-of-band)**

Token is at `/home/dustfeather/.cf-token` (P1, verified). Build the **gitignored** secret from it deterministically — never echo the token, never `git add secrets/cf-api-token.yaml`:

```bash
TOKEN="$(tr -d ' \n\r' < /home/dustfeather/.cf-token)"
sed "s|REPLACE_ME_CLOUDFLARE_API_TOKEN|${TOKEN}|" \
  secrets/cf-api-token.example > secrets/cf-api-token.yaml
unset TOKEN
kubectl apply -f secrets/cf-api-token.yaml
```

- [ ] **Step 2: Verify the secret exists and the ClusterIssuer does not (the failing test)**

Run: `kubectl -n cert-manager get secret cloudflare-api-token && kubectl get clusterissuer letsencrypt-cloudflare`
Expected: secret exists; `Error from server (NotFound)` for the clusterissuer.

- [ ] **Step 3: Write `manifests/10-clusterissuer-letsencrypt.yaml`**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cloudflare
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: dustfeather@gmail.com
    privateKeySecretRef:
      name: letsencrypt-cloudflare-account-key
    solvers:
      - dns01:
          cloudflare:
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
        selector:
          dnsZones:
            - itguys.ro
```

- [ ] **Step 4: Apply and verify the issuer is Ready (the passing test)**

Run: `kubectl apply -f manifests/10-clusterissuer-letsencrypt.yaml && kubectl wait --for=condition=Ready clusterissuer/letsencrypt-cloudflare --timeout=120s`
Expected: `clusterissuer.cert-manager.io/letsencrypt-cloudflare condition met`. If it does not become Ready, run `kubectl describe clusterissuer letsencrypt-cloudflare` — a registration error usually means a bad ACME email or the account key secret could not be created.

- [ ] **Step 5: Commit**

```bash
git add manifests/10-clusterissuer-letsencrypt.yaml
git commit -m "feat(nextcloud): Let's Encrypt Cloudflare DNS-01 ClusterIssuer"
```

---

## Task 4: Certificate for nextcloud.itguys.ro

**Files:**
- Create: `manifests/20-certificate.yaml`

- [ ] **Step 1: Verify the cert secret is absent (the failing test)**

Run: `kubectl -n nextcloud get secret nextcloud-tls`
Expected: `NotFound`.

- [ ] **Step 2: Write `manifests/20-certificate.yaml`**

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nextcloud-tls
  namespace: nextcloud
spec:
  secretName: nextcloud-tls
  duration: 2160h    # 90d
  renewBefore: 720h  # 30d — auto-renew, no manual action (design §3 TLS)
  privateKey:
    algorithm: ECDSA
    size: 256
  dnsNames:
    - nextcloud.itguys.ro
  issuerRef:
    name: letsencrypt-cloudflare
    kind: ClusterIssuer
```

- [ ] **Step 3: Apply and wait for issuance via DNS-01 (the passing test)**

```bash
kubectl apply -f manifests/20-certificate.yaml
kubectl -n nextcloud wait --for=condition=Ready certificate/nextcloud-tls --timeout=600s
```
Expected: `condition met` (DNS-01 propagation can take a few minutes). If it stalls, inspect: `kubectl -n nextcloud describe certificate nextcloud-tls` and `kubectl -n nextcloud get challenges,orders`. A challenge stuck `pending` almost always means the Cloudflare token lacks `Zone:DNS:Edit` on `itguys.ro`.

- [ ] **Step 4: Verify renewal is scheduled (design §7 acceptance)**

Run: `kubectl -n nextcloud get certificate nextcloud-tls -o jsonpath='{.status.notAfter} {.status.renewalTime}{"\n"}'`
Expected: two future timestamps, `renewalTime` ~30 days before `notAfter`.

- [ ] **Step 5: Commit**

```bash
git add manifests/20-certificate.yaml
git commit -m "feat(nextcloud): cert-manager Certificate for nextcloud.itguys.ro"
```

---

## Task 5: Standalone MariaDB

**Files:**
- Create: `manifests/30-mariadb.yaml`

**Prerequisite:** `secrets/nextcloud-db.yaml` applied.

- [ ] **Step 1: Apply the DB secret and verify MariaDB is absent (the failing test)**

```bash
cp secrets/nextcloud-db.example secrets/nextcloud-db.yaml
# edit -> real passwords
kubectl apply -f secrets/nextcloud-db.yaml
kubectl -n nextcloud get deploy nextcloud-mariadb
```
Expected: secret created; deployment `NotFound`.

- [ ] **Step 2: Write `manifests/30-mariadb.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-db
  namespace: nextcloud
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-path
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud-mariadb
  namespace: nextcloud
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels: { app: nextcloud-mariadb }
  template:
    metadata:
      labels: { app: nextcloud-mariadb }
    spec:
      nodeSelector:
        kubernetes.io/hostname: asus-laptop
      containers:
        - name: mariadb
          image: mariadb:11.4
          args:
            - --transaction-isolation=READ-COMMITTED
            - --character-set-server=utf8mb4
            - --collation-server=utf8mb4_general_ci
            - --innodb-file-per-table=1
            - --max-allowed-packet=256M
            - --skip-log-bin
          env:
            - name: MARIADB_ROOT_PASSWORD
              valueFrom: { secretKeyRef: { name: nextcloud-db, key: mariadb-root-password } }
            - name: MARIADB_DATABASE
              value: nextcloud
            - name: MARIADB_USER
              valueFrom: { secretKeyRef: { name: nextcloud-db, key: db-username } }
            - name: MARIADB_PASSWORD
              valueFrom: { secretKeyRef: { name: nextcloud-db, key: db-password } }
          ports:
            - { name: mysql, containerPort: 3306 }
          resources:
            requests: { cpu: 100m, memory: 512Mi }
          readinessProbe:
            exec:
              command: ["healthcheck.sh", "--connect", "--innodb_initialized"]
            initialDelaySeconds: 20
            periodSeconds: 10
          livenessProbe:
            exec:
              command: ["healthcheck.sh", "--connect"]
            initialDelaySeconds: 60
            periodSeconds: 15
          volumeMounts:
            - { name: data, mountPath: /var/lib/mysql }
      volumes:
        - name: data
          persistentVolumeClaim: { claimName: nextcloud-db }
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud-mariadb
  namespace: nextcloud
spec:
  selector: { app: nextcloud-mariadb }
  ports:
    - { name: mysql, port: 3306, targetPort: 3306 }
```

- [ ] **Step 3: Apply and verify Ready (the passing test)**

Run: `kubectl apply -f manifests/30-mariadb.yaml && kubectl -n nextcloud rollout status deploy/nextcloud-mariadb --timeout=180s`
Expected: `deployment "nextcloud-mariadb" successfully rolled out`.

- [ ] **Step 4: Verify DB connectivity with the Nextcloud user**

Run: `kubectl -n nextcloud exec deploy/nextcloud-mariadb -- sh -c 'mariadb -unextcloud -p"$MARIADB_PASSWORD" -e "SELECT 1;" nextcloud'`
Expected: a `1` result row, no auth error.

- [ ] **Step 5: Commit**

```bash
git add manifests/30-mariadb.yaml
git commit -m "feat(nextcloud): standalone MariaDB 11.4 (PVC+Deployment+Service)"
```

---

## Task 6: Standalone Valkey

**Files:**
- Create: `manifests/40-valkey.yaml`

**Prerequisite:** `secrets/valkey-auth.yaml` applied.

- [ ] **Step 1: Apply the Valkey secret and verify Valkey is absent (the failing test)**

```bash
cp secrets/valkey-auth.example secrets/valkey-auth.yaml
# edit -> real password
kubectl apply -f secrets/valkey-auth.yaml
kubectl -n nextcloud get deploy valkey
```
Expected: secret created; deployment `NotFound`.

- [ ] **Step 2: Write `manifests/40-valkey.yaml`**

Note: no persistence — Valkey holds only cache + transient file locks (Nextcloud re-acquires locks on reconnect). `emptyDir` is intentional (design §3: "Cache / file-locking").

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: valkey
  namespace: nextcloud
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels: { app: valkey }
  template:
    metadata:
      labels: { app: valkey }
    spec:
      nodeSelector:
        kubernetes.io/hostname: asus-laptop
      containers:
        - name: valkey
          image: valkey/valkey:8-alpine
          command:
            - sh
            - -c
            - exec valkey-server --requirepass "$VALKEY_PASSWORD" --save "" --appendonly no --maxmemory 512mb --maxmemory-policy allkeys-lru
          env:
            - name: VALKEY_PASSWORD
              valueFrom: { secretKeyRef: { name: valkey-auth, key: redis-password } }
          ports:
            - { name: valkey, containerPort: 6379 }
          resources:
            requests: { cpu: 25m, memory: 64Mi }
          readinessProbe:
            exec:
              command: ["sh", "-c", "valkey-cli -a \"$VALKEY_PASSWORD\" ping | grep -q PONG"]
            initialDelaySeconds: 5
            periodSeconds: 10
          volumeMounts:
            - { name: data, mountPath: /data }
      volumes:
        - name: data
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: valkey
  namespace: nextcloud
spec:
  selector: { app: valkey }
  ports:
    - { name: valkey, port: 6379, targetPort: 6379 }
```

- [ ] **Step 3: Apply and verify Ready (the passing test)**

Run: `kubectl apply -f manifests/40-valkey.yaml && kubectl -n nextcloud rollout status deploy/valkey --timeout=120s`
Expected: `successfully rolled out`.

- [ ] **Step 4: Verify auth works**

Run: `kubectl -n nextcloud exec deploy/valkey -- sh -c 'valkey-cli -a "$VALKEY_PASSWORD" ping'`
Expected: `PONG` (a `NOAUTH` error means the secret/env wiring is wrong).

- [ ] **Step 5: Commit**

```bash
git add manifests/40-valkey.yaml
git commit -m "feat(nextcloud): standalone Valkey (cache + file locking, no persistence)"
```

---

## Task 7: Nextcloud data + backups PVCs

**Files:**
- Create: `manifests/50-nextcloud-pvcs.yaml`

- [ ] **Step 1: Verify PVCs are absent (the failing test)**

Run: `kubectl -n nextcloud get pvc nextcloud-data nextcloud-backups`
Expected: `NotFound` for both.

- [ ] **Step 2: Write `manifests/50-nextcloud-pvcs.yaml`**

Sizes from design §3 ("Storage"): data 300Gi, backups 100Gi. (local-path does not hard-enforce quota; sizes are nominal — actual growth is monitored against asus's ~420 GB free.)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-data
  namespace: nextcloud
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-path
  resources:
    requests:
      storage: 300Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-backups
  namespace: nextcloud
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-path
  resources:
    requests:
      storage: 100Gi
```

- [ ] **Step 3: Apply (binding is WaitForFirstConsumer — Pending is expected until a pod mounts it)**

Run: `kubectl apply -f manifests/50-nextcloud-pvcs.yaml && kubectl -n nextcloud get pvc`
Expected: both PVCs exist, STATUS `Pending` with `VOLUMEBINDINGMODE` WaitForFirstConsumer (this is correct, not an error — local-path binds on first consumer).

- [ ] **Step 4: Commit**

```bash
git add manifests/50-nextcloud-pvcs.yaml
git commit -m "feat(nextcloud): data (300Gi) + backups (100Gi) PVCs"
```

---

## Task 8: Nextcloud Helm release

**Files:**
- Create: `helm/nextcloud-values.yaml`

**Prerequisite:** `secrets/nextcloud-admin.yaml` applied.

- [ ] **Step 1: Apply the admin secret and verify the release is absent (the failing test)**

```bash
cp secrets/nextcloud-admin.example secrets/nextcloud-admin.yaml
# edit -> real admin password
kubectl apply -f secrets/nextcloud-admin.yaml
helm -n nextcloud list | grep nextcloud || echo "no release yet (expected)"
```
Expected: secret created; no `nextcloud` release.

- [ ] **Step 2: Write `helm/nextcloud-values.yaml`**

```yaml
# Apache image (plain HTTP on :80, Service :8080). TLS is terminated by the
# separate nginx front-proxy (Task 9). Chart nginx sidecar intentionally OFF.
nginx:
  enabled: false

nextcloud:
  host: nextcloud.itguys.ro
  existingSecret:
    enabled: true
    secretName: nextcloud-admin
    usernameKey: nextcloud-username
    passwordKey: nextcloud-password
  trustedDomains:
    - nextcloud.itguys.ro
  extraEnv:
    - name: TRUSTED_PROXIES
      value: "10.42.0.0/16"          # in-cluster pod CIDR (front-proxy source)
    - name: OVERWRITEPROTOCOL
      value: "https"
    - name: OVERWRITEHOST
      value: "nextcloud.itguys.ro"            # served on default :443 via proxy hostPort (Task 9)
    - name: OVERWRITECLIURL
      value: "https://nextcloud.itguys.ro"
    - name: PHP_MEMORY_LIMIT
      value: "1024M"
    - name: PHP_UPLOAD_LIMIT
      value: "10G"

internalDatabase:
  enabled: false

externalDatabase:
  enabled: true
  type: mysql
  host: nextcloud-mariadb:3306
  database: nextcloud
  existingSecret:
    enabled: true
    secretName: nextcloud-db
    usernameKey: db-username
    passwordKey: db-password

# Bundled bitnami subcharts OFF (we run standalone MariaDB + Valkey).
mariadb:
  enabled: false
postgresql:
  enabled: false
redis:
  enabled: false

externalRedis:
  enabled: true
  host: valkey
  port: "6379"
  existingSecret:
    enabled: true
    secretName: valkey-auth
    passwordKey: redis-password

persistence:
  enabled: true
  existingClaim: nextcloud-data
  accessMode: ReadWriteOnce

# Nextcloud background jobs (separate from the Task-11 backup CronJob).
cronjob:
  enabled: true
  type: sidecar

service:
  type: ClusterIP
  port: 8080

# Pin the whole stack to asus (design §3 invariant).
nodeSelector:
  kubernetes.io/hostname: asus-laptop

startupProbe:
  enabled: true

# No memory limit: PHP_MEMORY_LIMIT=1024M per worker; burst is acceptable on
# asus (single/small-team use). Add limits.memory if the node shows OOM pressure.
resources:
  requests:
    cpu: 250m
    memory: 512Mi
```

- [ ] **Step 3: Install the chart (pinned version)**

```bash
helm repo add nextcloud https://nextcloud.github.io/helm/
helm repo update
helm install nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --version 9.1.0 \
  -f helm/nextcloud-values.yaml
```
Expected: `STATUS: deployed`.

- [ ] **Step 4: Verify rollout and PVC binding (the passing test)**

```bash
kubectl -n nextcloud rollout status deploy/nextcloud --timeout=600s
kubectl -n nextcloud get pvc nextcloud-data
```
Expected: `successfully rolled out`; `nextcloud-data` now `Bound`. First boot runs the Nextcloud installer (can take minutes). If the pod crash-loops, check `kubectl -n nextcloud logs deploy/nextcloud -c nextcloud` — the usual cause is a DB password mismatch between `nextcloud-db` secret and the MariaDB Deployment.

- [ ] **Step 5: Verify Nextcloud is installed and using Redis/Valkey for locking**

```bash
kubectl -n nextcloud exec deploy/nextcloud -c nextcloud -- php occ status
kubectl -n nextcloud exec deploy/nextcloud -c nextcloud -- php occ config:system:get memcache.locking
```
Expected: `installed: true`; `memcache.locking` → `\OC\Memcache\Redis`.

- [ ] **Step 6: Commit**

```bash
git add helm/nextcloud-values.yaml
git commit -m "feat(nextcloud): Nextcloud chart 9.1.0 (external MariaDB+Valkey, asus-pinned)"
```

---

## Task 9: nginx TLS front-proxy (hostPort :443)

**Files:**
- Create: `manifests/60-nginx-tls-proxy.yaml`

- [ ] **Step 1: Verify the proxy + Service are absent (the failing test)**

Run: `kubectl -n nextcloud get deploy nginx-tls-proxy svc nextcloud-https`
Expected: `NotFound` for both.

- [ ] **Step 2: Write `manifests/60-nginx-tls-proxy.yaml`**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-tls-proxy
  namespace: nextcloud
data:
  nginx.conf: |
    worker_processes auto;
    events { worker_connections 1024; }
    http {
      map $http_upgrade $connection_upgrade { default upgrade; '' close; }
      server {
        listen 443 ssl;
        http2 on;
        server_name nextcloud.itguys.ro;

        ssl_certificate     /tls/tls.crt;
        ssl_certificate_key /tls/tls.key;
        ssl_protocols TLSv1.2 TLSv1.3;

        client_max_body_size 10G;
        client_body_timeout 300s;
        proxy_read_timeout 3600s;
        proxy_request_buffering off;

        # CalDAV/CardDAV well-known redirects (Nextcloud security scan).
        # Use $http_host (the client's Host verbatim) so the redirect targets
        # whatever host[:port] the client used — robust if a non-default port
        # is ever introduced; correct for DAVx5/Apple/Thunderbird autodiscovery.
        location = /.well-known/carddav { return 301 https://$http_host/remote.php/dav; }
        location = /.well-known/caldav  { return 301 https://$http_host/remote.php/dav; }

        location / {
          proxy_pass http://nextcloud:8080;
          proxy_set_header Host              $host;
          proxy_set_header X-Real-IP         $remote_addr;
          proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Host  $http_host;
          proxy_set_header X-Forwarded-Proto https;
          proxy_set_header Upgrade           $http_upgrade;
          proxy_set_header Connection        $connection_upgrade;
        }
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-tls-proxy
  namespace: nextcloud
spec:
  replicas: 1
  strategy:
    type: Recreate          # single owner of host :443 — old pod must free it before new binds
  selector:
    matchLabels: { app: nginx-tls-proxy }
  template:
    metadata:
      labels: { app: nginx-tls-proxy }
    spec:
      nodeSelector:
        kubernetes.io/hostname: asus-laptop
      containers:
        - name: nginx
          image: nginx:1.27-alpine
          # Wrapper: nginx as PID 1 (exec) + background watcher that runs
          # `nginx -s reload` when cert-manager rotates nextcloud-tls in-place
          # (~every 60d). Without this the pod serves the OLD cert until a
          # restart, silently breaking design §3/§7 "auto-renew, no manual action".
          command:
            - /bin/sh
            - -c
            - |
              LAST=$(sha256sum /tls/tls.crt 2>/dev/null | awk '{print $1}')
              ( while true; do
                  sleep 21600
                  CUR=$(sha256sum /tls/tls.crt 2>/dev/null | awk '{print $1}')
                  if [ -n "$CUR" ] && [ "$CUR" != "$LAST" ]; then
                    echo "[cert-reload] nextcloud-tls changed, reloading nginx"
                    nginx -s reload
                    LAST="$CUR"
                  fi
                done ) &
              exec nginx -g 'daemon off;'
          ports:
            # hostPort 443: pod (pinned to asus, replicas:1) binds the asus
            # host's :443 directly, so the URL is https://nextcloud.itguys.ro
            # (no NodePort). nginx master runs as root → may bind privileged 443.
            - { name: https, containerPort: 443, hostPort: 443 }
          resources:
            requests: { cpu: 25m, memory: 32Mi }
          readinessProbe:
            tcpSocket: { port: 443 }
            initialDelaySeconds: 5
            periodSeconds: 10
          volumeMounts:
            - { name: conf, mountPath: /etc/nginx/nginx.conf, subPath: nginx.conf }
            - { name: tls,  mountPath: /tls, readOnly: true }
      volumes:
        - name: conf
          configMap: { name: nginx-tls-proxy }
        - name: tls
          secret: { secretName: nextcloud-tls }
---
apiVersion: v1
kind: Service
metadata:
  name: nextcloud-https
  namespace: nextcloud
spec:
  type: ClusterIP          # external access is via the pod's hostPort 443 on asus; this Service is in-cluster only
  selector: { app: nginx-tls-proxy }
  ports:
    - name: https
      port: 443
      targetPort: 443
```

- [ ] **Step 3: Apply and verify rollout (the passing test)**

Run: `kubectl apply -f manifests/60-nginx-tls-proxy.yaml && kubectl -n nextcloud rollout status deploy/nginx-tls-proxy --timeout=120s`
Expected: `successfully rolled out`. If nginx crash-loops with an SSL cert error, Task 4's `nextcloud-tls` secret is missing or not yet populated — re-check `kubectl -n nextcloud get secret nextcloud-tls`.

- [ ] **Step 4: Verify TLS + Nextcloud reachable in-cluster via the proxy Service**

```bash
kubectl -n nextcloud exec deploy/nextcloud -c nextcloud -- \
  sh -c 'curl -sk -H "Host: nextcloud.itguys.ro" https://nextcloud-https/status.php'
```
Expected: JSON containing `"installed":true` and `"productname":"Nextcloud"`.

- [ ] **Step 5: Commit**

```bash
git add manifests/60-nginx-tls-proxy.yaml
git commit -m "feat(nextcloud): nginx TLS front-proxy (hostPort :443)"
```

---

## Task 10: DNS + end-to-end verification

**Prerequisite:** P2 done (`nextcloud.itguys.ro` A → `100.96.0.2`, DNS-only). P3 recommended for the phone check.

- [ ] **Step 1: Verify DNS resolves to the asus Mesh IP (the failing test → passing once P2 is done)**

Run: `dig +short nextcloud.itguys.ro`
Expected: `100.96.0.2`. (If empty, P2 is not done or not propagated.)

- [ ] **Step 2: Verify a valid public cert is served end-to-end from a Mesh client**

Run (from this WSL session, a Mesh participant):
```bash
curl -sS -o /dev/null -w '%{http_code} %{ssl_verify_result}\n' https://nextcloud.itguys.ro/status.php
echo | openssl s_client -connect 100.96.0.2:443 -servername nextcloud.itguys.ro 2>/dev/null | openssl x509 -noout -issuer -dates
```
Expected: `200 0` (HTTP 200, TLS verify OK = 0); issuer contains `Let's Encrypt`, valid date range. **No `-k` needed** — this is the §7 "valid cert, no warnings" check.

- [ ] **Step 3: Verify web login works**

In a browser on a Mesh machine open `https://nextcloud.itguys.ro`, log in with the admin credentials from `secrets/nextcloud-admin.yaml`. Expected: dashboard loads, no certificate warning, no "untrusted domain" error.

- [ ] **Step 4: Verify the Android app (design §7)**

On the WARP-enrolled phone (P3), install the Nextcloud Android app, server URL `https://nextcloud.itguys.ro`, log in. Expected: files sync; add the account in DAVx⁵/built-in to confirm CalDAV/CardDAV.

- [ ] **Step 5: Commit (documentation only — capture the verified access URL)**

No artifact change in this task; if any notes were added to README, commit them, otherwise skip. Record the verified URL `https://nextcloud.itguys.ro` in Task 12's README update.

---

## Task 11: Nightly backup CronJob

**Prerequisite:** none human — acer host-side prep is automated in Step 1 (remote sudo on `acer-laptop` is available, see Conventions).

**Files:**
- Create: `manifests/70-backup-cronjob.yaml`

- [ ] **Step 1: Generate the backup key, provision acer over remote sudo, apply the SSH secret (the failing test)**

```bash
# 1. Dedicated keypair (NOT a personal key)
ssh-keygen -t ed25519 -N '' -f /tmp/nc-backup -C nextcloud-backup

# 2. Provision the acer backup target over Mesh SSH (remote access available)
PUB="$(cat /tmp/nc-backup.pub)"
ssh dustfeather@100.96.0.4 'mkdir -p ~/backup/nextcloud/data ~/backup/nextcloud/db && chmod 700 ~/backup ~/backup/nextcloud'
ssh dustfeather@100.96.0.4 "install -d -m700 ~/.ssh && touch ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && grep -qxF \"$PUB\" ~/.ssh/authorized_keys || printf '%s\n' \"$PUB\" >> ~/.ssh/authorized_keys"

# 3. Verify the DEDICATED key reaches acer non-interactively
ssh -i /tmp/nc-backup -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new \
  dustfeather@100.96.0.4 'echo acer-backup-ok; ls -ld ~/backup/nextcloud/data ~/backup/nextcloud/db'

# 4. Build the gitignored secret DETERMINISTICALLY from the keyfile (no manual
#    paste, no REPLACE_ME failure mode). Key name id_ed25519 matches the CronJob.
kubectl create secret generic backup-ssh -n nextcloud \
  --from-file=id_ed25519=/tmp/nc-backup \
  --dry-run=client -o yaml > secrets/backup-ssh.yaml
# Guard (carry-forward, Task 1 review): never apply a placeholder/empty key.
grep -q 'REPLACE_ME' secrets/backup-ssh.yaml && { echo "ERROR: placeholder in backup-ssh.yaml"; exit 1; } || true
grep -q 'id_ed25519' secrets/backup-ssh.yaml || { echo "ERROR: id_ed25519 key missing"; exit 1; }
kubectl apply -f secrets/backup-ssh.yaml
kubectl -n nextcloud get cronjob nextcloud-backup
```
Expected: step 3 prints `acer-backup-ok` and the two backup dirs; secret created; cronjob `NotFound`. If an `ssh` line fails with a connection timeout, retry it once (transient Mesh connect — see Conventions) before treating it as a real failure.

- [ ] **Step 2: Write `manifests/70-backup-cronjob.yaml`**

Retention: **7 daily** (user decision). Nightly 03:00. Single-transaction dump (consistent, no maintenance mode — see Deviations §2).

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nextcloud-backup
  namespace: nextcloud
spec:
  schedule: "0 3 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      template:
        spec:
          restartPolicy: Never
          nodeSelector:
            kubernetes.io/hostname: asus-laptop
          containers:
            - name: backup
              image: alpine:3.21
              env:
                - name: MARIADB_ROOT_PASSWORD
                  valueFrom: { secretKeyRef: { name: nextcloud-db, key: mariadb-root-password } }
              resources:
                requests: { cpu: 100m, memory: 256Mi }
              command:
                - sh
                - -euc
                - |
                  apk add --no-cache mariadb-client rsync openssh-client gzip coreutils >/dev/null
                  TS=$(date +%F)
                  mkdir -p /backups/db
                  # Dump to a temp file FIRST: POSIX sh has no pipefail, so
                  # `mariadb-dump | gzip` would mask a dump failure (gzip exits 0
                  # on empty stdin → a valid but EMPTY .gz shipped as "success").
                  # Writing to a file makes `set -e` abort on dump failure.
                  echo "[backup] dumping DB"
                  # NOTE: uses MariaDB root (homelab trade-off; a least-priv backup
                  # user would need SELECT,LOCK TABLES,SHOW VIEW,EVENT,TRIGGER).
                  mariadb-dump --single-transaction --quick --routines --triggers \
                    -h nextcloud-mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" nextcloud > /tmp/dump.sql
                  SZ=$(wc -c < /tmp/dump.sql)
                  [ "$SZ" -gt 1024 ] || { echo "[backup] FATAL: dump only $SZ bytes — aborting"; exit 1; }
                  gzip -c /tmp/dump.sql > "/backups/db/db-$TS.sql.gz"
                  rm -f /tmp/dump.sql
                  gzip -t "/backups/db/db-$TS.sql.gz"
                  echo "[backup] capturing config.php"
                  cp /nc-data/config/config.php "/backups/db/config-$TS.php"
                  echo "[backup] pruning LOCAL copies older than 7 days (acer keeps full dump history by design — §4 recovery copy)"
                  find /backups/db -name 'db-*.sql.gz' -mtime +7 -delete
                  find /backups/db -name 'config-*.php' -mtime +7 -delete
                  # Guard the destructive mirror: if the data PVC mounted empty,
                  # an `rsync --delete` would wipe the acer recovery copy. config.php
                  # was just read above, so its absence means /nc-data is bad.
                  [ -f /nc-data/config/config.php ] || { echo "[backup] FATAL: /nc-data looks empty — refusing to rsync --delete"; exit 1; }
                  echo "[backup] rsync data + latest dump off-node to acer-laptop"
                  install -m600 /ssh/id_ed25519 /tmp/id
                  SSHOPT="-i /tmp/id -o StrictHostKeyChecking=accept-new -o UserKnownHostsFile=/tmp/khosts"
                  rsync -a --delete -e "ssh $SSHOPT" \
                    /nc-data/ dustfeather@100.96.0.4:~/backup/nextcloud/data/
                  rsync -a -e "ssh $SSHOPT" \
                    "/backups/db/db-$TS.sql.gz" "/backups/db/config-$TS.php" \
                    dustfeather@100.96.0.4:~/backup/nextcloud/db/
                  echo "[backup] done"
              volumeMounts:
                - { name: ncdata,  mountPath: /nc-data, readOnly: true }
                - { name: backups, mountPath: /backups }
                - { name: ssh,     mountPath: /ssh, readOnly: true }
          volumes:
            - name: ncdata
              persistentVolumeClaim: { claimName: nextcloud-data }
            - name: backups
              persistentVolumeClaim: { claimName: nextcloud-backups }
            - name: ssh
              secret: { secretName: backup-ssh, defaultMode: 0400 }
```

- [ ] **Step 3: Apply and trigger a manual run (the passing test)**

```bash
kubectl apply -f manifests/70-backup-cronjob.yaml
kubectl -n nextcloud create job --from=cronjob/nextcloud-backup nextcloud-backup-manual
kubectl -n nextcloud wait --for=condition=complete job/nextcloud-backup-manual --timeout=900s
kubectl -n nextcloud logs job/nextcloud-backup-manual
```
Expected: job completes; log ends with `[backup] done`. If rsync fails with a host-key/permission error, Step 1's acer provisioning is incomplete (pubkey not authorized on acer or `~/backup/nextcloud/` missing).

- [ ] **Step 3b: Protect the backups PV from accidental deletion (CARRY-FORWARD from Task 7 review — REQUIRED, not over-build)**

The manual job mounted `nextcloud-backups`, so it is now Bound (local-path default reclaimPolicy is Delete):
```bash
PV=$(kubectl -n nextcloud get pvc nextcloud-backups -o jsonpath='{.spec.volumeName}')
kubectl patch pv "$PV" -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl get pv "$PV" -o jsonpath='{.metadata.name} {.spec.persistentVolumeReclaimPolicy}{"\n"}'
```
Expected: prints the PV name and `Retain`. (Operational cluster patch, no repo artifact; report it for Task 12's README note.)

- [ ] **Step 4: Verify the restore artifacts exist locally AND on acer**

```bash
kubectl -n nextcloud exec deploy/nextcloud-mariadb -- sh -c 'ls -la /dev/null' >/dev/null  # noop guard
kubectl -n nextcloud run bkcheck --rm -i --restart=Never --image=busybox \
  --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"asus-laptop"},"containers":[{"name":"c","image":"busybox","command":["sh","-c","ls -la /b/db"],"volumeMounts":[{"name":"b","mountPath":"/b"}]}],"volumes":[{"name":"b","persistentVolumeClaim":{"claimName":"nextcloud-backups"}}]}}'
ssh dustfeather@100.96.0.4 'ls -la ~/backup/nextcloud/db/ && du -sh ~/backup/nextcloud/data/'
```
Expected: a `db-<date>.sql.gz` and `config-<date>.php` in the backups PVC and on acer; `data/` present on acer with non-zero size.

- [ ] **Step 5: Sanity-check the dump is restorable (gzip integrity + SQL header)**

```bash
kubectl -n nextcloud run bkverify --rm -i --restart=Never --image=alpine:3.21 \
  --overrides='{"spec":{"nodeSelector":{"kubernetes.io/hostname":"asus-laptop"},"containers":[{"name":"c","image":"alpine:3.21","command":["sh","-c","gzip -t /b/db/db-*.sql.gz && zcat /b/db/db-*.sql.gz | head -5"],"volumeMounts":[{"name":"b","mountPath":"/b"}]}],"volumes":[{"name":"b","persistentVolumeClaim":{"claimName":"nextcloud-backups"}}]}}'
kubectl -n nextcloud delete job nextcloud-backup-manual
```
Expected: `gzip -t` exits clean; head shows a MariaDB dump header (`-- MariaDB dump ...`).

- [ ] **Step 6: Commit**

```bash
git add manifests/70-backup-cronjob.yaml
git commit -m "feat(nextcloud): nightly backup CronJob (7d local + rsync to acer)"
```

---

## Task 12: Post-deploy hardening + README apply order

**Files:**
- Modify: `README.md` (replace the "Apply order" TBD section)

- [ ] **Step 1: Run Nextcloud's own security/setup checks (design §7)**

```bash
kubectl -n nextcloud exec deploy/nextcloud -c nextcloud -- php occ setupchecks
```
Expected: no critical errors. Common acceptable info: phone-region not set. If "reverse proxy headers" or "HTTPS" warnings appear, recheck the `OVERWRITE*`/`TRUSTED_PROXIES` env from Task 8.

- [ ] **Step 2: Enable two-factor app (design §3 security model)**

```bash
kubectl -n nextcloud exec deploy/nextcloud -c nextcloud -- php occ app:enable twofactor_totp
```
Expected: `twofactor_totp enabled`. (Per-user 2FA enforcement is an admin policy in Settings → Security; note this for the operator — it cannot be forced without each user enrolling.)

- [ ] **Step 3: Replace the "Apply order" section in `README.md`**

Replace the block under `## Apply order (filled in by the implementation plan)` (currently the `TBD ...` paragraph) with:

```markdown
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
```

- [ ] **Step 4: Update the "Status" section in `README.md`**

Replace the `## Status` body with: `Deployed. Plan executed: docs/superpowers/plans/2026-05-17-nextcloud-k3s-deployment.md. Access https://nextcloud.itguys.ro (Mesh only).`

- [ ] **Step 4b: Add an "Operational dependencies" section to `README.md`** (carry-forwards: Gateway Do-Not-Inspect + PV Retain). Append:

```markdown
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
- **PV reclaim policy = Retain** — the bound PVs for `nextcloud-data` and
  `nextcloud-backups` were patched to `persistentVolumeReclaimPolicy: Retain`
  (local-path defaults to `Delete`), so an accidental `kubectl delete pvc`
  does not wipe the hostPath. Disk-loss recovery is still the nightly
  acer-laptop rsync (design §4); these PVs are single-disk on asus.
```

- [ ] **Step 4c: Refresh the now-stale sub-`README.md`s** (they still say "None yet (design only)"):
  - `helm/README.md`: state it now holds `cert-manager-values.yaml` + `nextcloud-values.yaml` (committed, secret-free).
  - `manifests/README.md`: state it now holds `00-namespace`, `10-clusterissuer-letsencrypt`, `20-certificate`, `30-mariadb`, `40-valkey`, `50-nextcloud-pvcs`, `60-nginx-tls-proxy`, `70-backup-cronjob` (applied; apply order in root README).
  - `secrets/README.md`: keep the policy text; update the closing line to note the `*.example` templates now exist and the real `*.yaml` are applied out-of-band & gitignored.

- [ ] **Step 4d: De-stale `CLAUDE.md` + root `README.md` Layout/Secrets (Task 12 code-review carry-forward — these operator docs still describe pre-deploy state)**

In `CLAUDE.md`, replace the `**Current state: design-only.** … the "Apply order" in the root `README.md`.` paragraph with:
```
**Current state: deployed.** The implementation plan
(`docs/superpowers/plans/2026-05-17-nextcloud-k3s-deployment.md`) has been written
and executed; `helm/`, `manifests/`, and `secrets/*.example` are populated and the
stack is live on the cluster (`https://nextcloud.itguys.ro`, Mesh-only, :443 via the
proxy hostPort). The root `README.md` "Apply order" and "Operational dependencies"
sections are the operator entry point; re-applying is manual (`helm upgrade` /
`kubectl apply`) per the versioned-imperative model below.
```

In `README.md` Layout block, replace the `helm/` and `manifests/` lines with:
```
helm/        Helm values (committed, secret-free): cert-manager + nextcloud
manifests/   raw k8s YAML: namespace, cert-manager issuer + certificate,
             MariaDB, Valkey, PVCs, nginx TLS proxy (hostPort :443), backup CronJob
```

In `README.md` "Secrets this deployment needs" list, replace the two bullets with:
```
- Cloudflare API token for cert-manager DNS-01 (`Zone:DNS:Edit` +
  `Zone:Zone:Read` on `itguys.ro`) — `secrets/cf-api-token`.
- Nextcloud admin password — `secrets/nextcloud-admin`.
- MariaDB root + nextcloud DB passwords — `secrets/nextcloud-db`.
- Valkey password — `secrets/valkey-auth`.
- Dedicated backup SSH private key (rsync to acer) — `secrets/backup-ssh`.
```

- [ ] **Step 5: Commit**

```bash
git add README.md helm/README.md manifests/README.md secrets/README.md
git commit -m "docs(nextcloud): finalize README — apply order (:443), status, operational deps, sub-READMEs"
```
(If Step 4d is done as a follow-up after Step 5's commit, use a second commit:
`git add CLAUDE.md README.md && git commit -m "docs(nextcloud): de-stale CLAUDE.md + README Layout/Secrets to deployed state"`.)

---

## Self-review (completed during planning)

- **Spec coverage:** §3 components → Tasks 5,6,8,9 + cert-manager 2–4; §3 storage → Tasks 5,7; §3 access (hostPort 443/DNS/overwrite) → Tasks 8,9,10; §3 TLS auto-renew → Task 4 (`renewBefore: 720h`, verified §7); §4 backups → Task 11 (acer host prep automated in 11.1 per the 2026-05-17 remote-sudo amendment); §5 prereqs → P1–P3 human + former P4 automated; §7 acceptance → Tasks 4,8,10,11,12; §8 deferred (Collabora/Photos) → intentionally omitted. No gaps.
- **Placeholder scan:** every YAML/command is concrete; the only `REPLACE_ME` tokens are inside `secrets/*.example` templates by design (real values are out-of-band, gitignored).
- **Type/name consistency:** secret names (`cloudflare-api-token`, `nextcloud-admin`, `nextcloud-db`, `valkey-auth`, `backup-ssh`, `nextcloud-tls`), Service DNS (`nextcloud-mariadb:3306`, `valkey:6379`, `nextcloud:8080`, `nextcloud-https`), PVCs (`nextcloud-data`, `nextcloud-db`, `nextcloud-backups`) and the `kubernetes.io/hostname: asus-laptop` selector are consistent across all tasks.
