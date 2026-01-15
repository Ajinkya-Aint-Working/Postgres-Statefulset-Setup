# 📘 PostgreSQL on Kubernetes (GKE)

**Persistence · Connection Control · Snapshots · Backups**

This document explains how to deploy **PostgreSQL as a StatefulSet** on Kubernetes running on **Google Kubernetes Engine (GKE)**, with:

- Persistent storage using **GCP Persistent Disk**
- Connection restrictions using `pg_hba.conf`
- Automated CSI **VolumeSnapshots**
- Snapshot retention (7 days)
- Clean RBAC model (1 ServiceAccount / Role / RoleBinding per namespace)
- Fully compatible with **GKE Autopilot** and **Argo CD**

---

## 🧠 Architecture Overview

### Key Principles

- One database per namespace  
- StatefulSet + PVC for persistence  
- `pg_hba.conf` controls who can connect  
- Snapshots are automated via CronJobs  
- No ClusterRole required  
- Runtime snapshots are **not managed by Argo CD**

---

## 1️⃣ PostgreSQL Deployment  
### StatefulSet + Headless Service

### Headless Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: solvox-postgres
  namespace: default
spec:
  clusterIP: None
  selector:
    app: solvox-postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
```

### PostgreSQL StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: solvox-postgres
  namespace: default
spec:
  serviceName: solvox-postgres
  replicas: 1
  selector:
    matchLabels:
      app: solvox-postgres
  template:
    metadata:
      labels:
        app: solvox-postgres
    spec:
      securityContext:
        fsGroup: 999

      initContainers:
        - name: init-chown
          image: busybox
          command: ["sh", "-c", "chown -R 999:999 /var/lib/postgresql/data || true"]
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data

      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
              name: postgres

          env:
            - name: POSTGRES_USER
              value: "solvox_user"
            - name: POSTGRES_PASSWORD
              value: "1234"
            - name: POSTGRES_DB
              value: "solvoxai"
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata

          args:
            - "-c"
            - "listen_addresses=*"
            - "-c"
            - "hba_file=/etc/postgresql/pg_hba.conf"

          resources:
            requests:
              cpu: 250m
              memory: 512Mi

          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
            - name: postgres-hba
              mountPath: /etc/postgresql/pg_hba.conf
              subPath: pg_hba.conf

      volumes:
        - name: postgres-hba
          configMap:
            name: postgres-hba-config

  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard-rwo
        resources:
          requests:
            storage: 1Gi
```

---

## 2️⃣ Connection Restriction (`pg_hba.conf`)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-hba-config
  namespace: default
data:
  pg_hba.conf: |
    local   all       all                    trust
    host    all       all   10.0.0.0/8        md5
    host    all       all   192.168.0.0/16    md5
    host    all       all   0.0.0.0/0         reject
```

Verify:
```sql
SHOW hba_file;
SELECT * FROM pg_hba_file_rules;
```

---

## 3️⃣ Snapshot Infrastructure (Cluster-wide)

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: gke-pd-snapshot-class
driver: pd.csi.storage.gke.io
deletionPolicy: Delete
```

---

## 4️⃣ RBAC (Per Namespace)

ServiceAccount, Role, RoleBinding (one-time per namespace).

---

## 5️⃣ Snapshot Creation CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-snapshot
  namespace: default
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: db-backup-sa
          restartPolicy: OnFailure
          containers:
            - name: snapshot
              image: bitnami/kubectl
              command:
                - /bin/sh
                - -c
                - |
                  kubectl apply -f - <<EOF
                  apiVersion: snapshot.storage.k8s.io/v1
                  kind: VolumeSnapshot
                  metadata:
                    name: solvox-postgres-snap-$(date +%F)
                    namespace: default
                  spec:
                    volumeSnapshotClassName: gke-pd-snapshot-class
                    source:
                      persistentVolumeClaimName: postgres-data-solvox-postgres-0
                  EOF
```

---

## 6️⃣ Snapshot Cleanup (7-day Retention)

```yamlapiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-snapshot-cleanup
  namespace: default
spec:
  schedule: "30 4 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: db-backup-sa
          restartPolicy: OnFailure
          containers:
            - name: cleanup
              image: bitnami/kubectl
              command:
                - /bin/sh
                - -c
                - |
                  kubectl get volumesnapshot \
                    -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.metadata.creationTimestamp}{"\n"}{end}' \
                  | while read name ts; do
                      if [ $(date -d "$ts" +%s) -lt $(date -d "7 days ago" +%s) ]; then
                        kubectl delete volumesnapshot "$name"
                      fi
                    done

```

---

## 7️⃣ Where to Find Snapshots

- Kubernetes: `kubectl get volumesnapshot -n default`
- GCP Console: Compute Engine → Snapshots

---

## 8️⃣ Restore Flow (High-level)

VolumeSnapshot → New PVC → Attach to StatefulSet → PostgreSQL WAL replay → Restored

---

## 9️⃣ Argo CD Notes

- Do NOT commit VolumeSnapshot YAMLs
- CronJobs are Git-managed
- Snapshots are runtime resources

---


