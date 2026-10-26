# Day 19 — Kubernetes Storage & StatefulSets

Part of my **DevOps Winter Arc — 90 Days Challenge**.

## Objective

Deploy PostgreSQL using a Kubernetes StatefulSet and persistent storage, then verify that a database record survives Pod deletion and recreation.

## Technologies

- Kubernetes
- StatefulSet
- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)
- StorageClass
- Headless Service
- PostgreSQL 16 Alpine
- kubectl
- WSL Ubuntu

## Architecture

```text
StatefulSet: pg
      |
      v
Pod: pg-0
      |
      v
PVC: data-pg-0 (1Gi)
      |
      v
PV: dynamically provisioned
      |
      v
StorageClass: standard
      |
      v
Local-path provisioner
```

The Headless Service `pg-headless` provides stable network identity for the StatefulSet Pod.

## Project files

- `pg-headless.yaml` — Headless Service manifest.
- `postgres-statefulset.yaml` — PostgreSQL StatefulSet and volume claim template.
- `screenshots/` — Evidence collected during the lab.

## Implementation

### 1. Inspect storage

```bash
kubectl get storageclass
kubectl get pv
kubectl get pvc -A
```

The cluster's default StorageClass was `standard`, using the `rancher.io/local-path` provisioner.

### 2. Create the namespace and Secret

```bash
kubectl create namespace day19-storage

kubectl create secret generic pg-secret \
  --from-literal='POSTGRES_PASSWORD=REPLACE_WITH_DUMMY_LAB_PASSWORD' \
  -n day19-storage
```

Replace the placeholder with a disposable lab password when running the command. Never commit the actual password.

### 3. Deploy the manifests

```bash
kubectl apply -f pg-headless.yaml
kubectl apply -f postgres-statefulset.yaml
```

### 4. Verify the resources

```bash
kubectl get svc,statefulset,pods,pvc -n day19-storage
kubectl get pv
```

Expected conditions:

- StatefulSet: `1/1` Ready.
- Pod: `pg-0`, `1/1 Running`.
- PVC: `data-pg-0`, `Bound`.
- PV: `Bound`.

### 5. Insert a test record

Connect to PostgreSQL:

```bash
kubectl exec -it -n day19-storage pg-0 -- \
  psql -U appuser -d winterarc
```

Run the SQL commands:

```sql
CREATE TABLE lab_notes (
    id INT PRIMARY KEY,
    note TEXT
);

INSERT INTO lab_notes
VALUES (1, 'data-survives-pod-delete');

SELECT * FROM lab_notes;
```

Exit with `\q`.

### 6. Test persistence after Pod deletion

Delete only the Pod:

```bash
kubectl delete pod pg-0 -n day19-storage
```

Wait for its replacement to become ready:

```bash
kubectl get pods -n day19-storage -w
```

Verify the record:

```bash
kubectl exec -n day19-storage pg-0 -- \
  psql -U appuser -d winterarc \
  -c "SELECT * FROM lab_notes;"
```

**Success condition:** The same row remains after Pod recreation.

## Storage concepts learned

- A PV represents a storage resource.
- A PVC requests storage and binds to a suitable PV.
- A StorageClass defines storage provisioning behavior.
- A StatefulSet provides stable Pod identities and persistent storage patterns.
- A Headless Service supports stable DNS identities for StatefulSet Pods.
- A PVC normally remains when only its Pod is deleted.
- Persistent storage does not replace backups or disaster recovery.

## PVC Pending troubleshooting

Two possible causes:

1. The requested StorageClass is missing or incorrectly named.
2. The storage provisioner is unhealthy or cannot create a volume.

Other causes include insufficient capacity or incompatible storage requirements. Inspect PVC Events before changing configuration.

```bash
kubectl describe pvc data-pg-0 -n day19-storage
kubectl get events -n day19-storage --sort-by=.lastTimestamp
```

## Evidence

Add screenshots showing:

- PVC `Bound` and PostgreSQL Pod readiness.
- The test row before and after Pod deletion.
- The storage chain and PV details.

## Cleanup

This is a disposable local lab. Save screenshots and evidence first.

```bash
kubectl delete namespace day19-storage
kubectl get pv
```

The PV reclaim policy in this cluster is `Delete`. Removing the namespace and PVC may delete the underlying lab storage.

## Key takeaway

A StatefulSet combined with persistent storage can preserve database data across Pod deletion and recreation, provided the same persistent volume is retained and reattached.
