# manifests/

Raw Kubernetes YAML applied to the cluster. Files (apply order in root README):

- `00-namespace.yaml`
- `10-clusterissuer-letsencrypt.yaml`
- `20-certificate.yaml`
- `30-mariadb.yaml`
- `40-valkey.yaml`
- `50-nextcloud-pvcs.yaml`
- `60-nginx-tls-proxy.yaml`
- `70-backup-cronjob.yaml`
