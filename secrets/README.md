# secrets/

Templates only. **No real secret values in git, ever.**

`.gitignore` permits only `README.md` and `*.example` / `*.template` here.
Anything else (real `*.yaml` secrets) is ignored.

Pattern:
1. `cp secrets/<name>.example secrets/<name>.yaml`
2. fill real values in `secrets/<name>.yaml` (gitignored)
3. `kubectl apply -f secrets/<name>.yaml`

`*.example` template files are committed (cf-api-token, nextcloud-admin,
nextcloud-db, valkey-auth, backup-ssh). The real `*.yaml` files are applied
out-of-band and gitignored — they are never committed.
