# CryptoCurrencyExchange Kubernetes manifests

These manifests intentionally create **one PostgreSQL replica** with persistent
storage. They do not provide PostgreSQL HA, a replica, an operator, Helm, or
off-cluster/object-storage backup replication.

## Required application correction

The supplied source hardcodes `SECRET_KEY` and treats the string `"False"` as
truthy. Before deployment, update `Exchange/Exchange/settings.py`:

```python
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
DEBUG = os.getenv("DEBUG", "False").lower() == "true"
ALLOWED_HOSTS = [host for host in os.getenv("ALLOWED_HOSTS", "").split(",") if host]
```

The `DATABASE_*` names in `configmap.yaml` and `secret.example.yaml` match the
names already read by the application. The container image field is deliberately
versioned rather than `latest`; replace `registry.example.com/cryptocurrency-exchange:1.0.0`
with the immutable image tag (and ideally registry digest) that you build and push.

## Deploy

1. Create the namespace:

   ```bash
   kubectl apply -f namespace.yaml
   ```

2. Create a real Secret without committing it. Copy the example outside Git,
   replace every `REPLACE_...` value, then apply it:

   ```bash
   cp secret.example.yaml /secure/location/secret.yaml
   kubectl apply -f /secure/location/secret.yaml
   ```

   `DATABASE_USER` and `POSTGRES_USER` must be identical; the same is true for
   their passwords.

3. Apply the ConfigMap and PostgreSQL, wait for the database, then migrate and
   deploy the application:

   ```bash
   kubectl apply -f configmap.yaml -f postgresql.yaml
   kubectl -n cryptocurrency-exchange rollout status deployment/postgresql --timeout=180s
   kubectl apply -f application.yaml
   kubectl -n cryptocurrency-exchange wait --for=condition=complete job/cryptocurrency-exchange-migrate --timeout=300s
   kubectl -n cryptocurrency-exchange rollout status deployment/cryptocurrency-exchange --timeout=180s
   kubectl apply -f backup.yaml
   ```

   `application.yaml` contains both the migration Job and Deployment. On later
   releases, delete/recreate the completed migration Job, or give each release a
   unique Job name, before applying a new migration.

## Access without an Ingress

```bash
kubectl -n cryptocurrency-exchange port-forward service/crypto-exchange 8000:8000
```

Open `http://127.0.0.1:8000`. For an Ingress later, add its hostname to
`ALLOWED_HOSTS` and use TLS at the ingress layer.

## Backup and restore

The CronJob runs daily at 02:00 cluster time, writes compressed PostgreSQL
custom-format dumps to the `postgresql-backups` PVC, verifies each archive with
`pg_restore --list`, and deletes backups older than 14 days. The backup location
is therefore the PV provisioned for `postgresql-backups`; configure an independent
StorageClass if the cluster offers one.

Run a manual backup immediately:

```bash
kubectl -n cryptocurrency-exchange create job --from=cronjob/cryptocurrency-exchange-postgresql-backup cryptocurrency-exchange-postgresql-backup-manual
kubectl -n cryptocurrency-exchange logs -f job/cryptocurrency-exchange-postgresql-backup-manual
```

`pg_restore --list` checks that the dump archive is readable, but a real restore
test should be run regularly against a separate temporary PostgreSQL database.
To restore, first scale the application to zero to stop writes, copy
`restore-job.template.yaml`, replace `BACKUP_FILE` with a dump filename, apply
the copy, and watch the Job:

```bash
kubectl -n cryptocurrency-exchange scale deployment/cryptocurrency-exchange --replicas=0
kubectl apply -f /secure/location/restore-job.yaml
kubectl -n cryptocurrency-exchange logs -f job/cryptocurrency-exchange-postgresql-restore
kubectl -n cryptocurrency-exchange scale deployment/cryptocurrency-exchange --replicas=2
```

The restore uses `--clean --if-exists`; it changes the target database and must
only be run after confirming the selected backup and a maintenance window.
