# PostgreSQL Backup in Kubernetes using Minikube

## ✅ Objective:

Create a **PostgreSQL database** using a **StatefulSet** in a Minikube Kubernetes cluster on Ubuntu, then implement a **daily backup** using a **CronJob** stored on a **PersistentVolume**, with **backup rotation**.

---

## 🧰 Environment Setup

* Ubuntu VM with:

  * Kubernetes installed
  * Minikube installed
  * Docker as runtime

```bash
minikube start --memory=3072 --cpus=2 --driver=docker
```

---

## 🛠️ Kubernetes Resource Setup

### 1. (Optional) Create a Namespace

```bash
kubectl create namespace postgres-ns
```

---

### 2. Secret for PostgreSQL credentials

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: postgres-ns
type: Opaque
stringData:
  username: postgres
  password: mypassword
```

---

### 3. PV and PVC for PostgreSQL Data

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
  namespace: postgres-ns
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/postgres
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: postgres-ns
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

### 4. StatefulSet for PostgreSQL

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: postgres-ns
spec:
  selector:
    matchLabels:
      app: postgres
  serviceName: "postgres"
  replicas: 1
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

---

### 5. Service to Expose PostgreSQL

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres-ns
spec:
  ports:
    - port: 5432
  selector:
    app: postgres
```

---

## 🛠️ Backup Setup

### 6. PV and PVC for Backups

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-backup-pv
  namespace: postgres-ns
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/data/postgres-backups
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-backup-pvc
  namespace: postgres-ns
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

---

### 7. ConfigMap for Backup Script

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-backup-script
  namespace: postgres-ns
data:
  backup.sh: |
    #!/bin/bash
    echo "Starting backup..."

    BACKUP_FILE="/backups/postgres_backup_$(date +%Y%m%d%H%M).sql"

    pg_dump -h postgres -U "$POSTGRES_USER" -d postgres > "$BACKUP_FILE"

    echo "✅ Backup created at $BACKUP_FILE"

    # Delete backups older than 7 days
    find /backups -type f -name "*.sql" -mtime +7 -exec rm {} \;

    echo "🧹 Old backups removed."
```

---

### 8. CronJob (Run Daily at 10 PM)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: postgres-ns
spec:
  schedule: "0 22 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:14
            command: ["/bin/bash", "/scripts/backup.sh"]
            env:
              - name: POSTGRES_USER
                valueFrom:
                  secretKeyRef:
                    name: postgres-secret
                    key: username
              - name: POSTGRES_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: postgres-secret
                    key: password
            volumeMounts:
              - name: backup-storage
                mountPath: /backups
              - name: backup-script
                mountPath: /scripts
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: postgres-backup-pvc
            - name: backup-script
              configMap:
                name: postgres-backup-script
          restartPolicy: OnFailure
```

---

## 🔢 Verification

```bash
# Check CronJobs
kubectl get cronjobs -n postgres-ns

# Check backups in logs
kubectl logs <job-pod-name> -n postgres-ns

# Check inside backup path in Minikube VM
minikube ssh
ls -lh /mnt/data/postgres-backups/
```

---

## 📅 Notes and Advice

* **You will not see the backup files on your local Ubuntu VM.**

  * The backups are created **inside the Minikube VM**.
  * To view or copy them to your local machine:

    ```bash
    minikube ssh
    ls /mnt/data/postgres-backups
    ```
  * To copy to host:

    ```bash
    minikube cp /mnt/data/postgres-backups/<file> ./
    ```

* **Advice:**

  * Be careful with PersistentVolumes and PersistentVolumeClaims.
  * Try to reduce unused or extra PVs to avoid confusion and interruptions while finishing the task.

---

## ❌ Cleanup (Optional)

```bash
kubectl delete namespace postgres-ns
```

---

### ✨ Great job completing this task! You now have an automated PostgreSQL backup running in Kubernetes on Minikube.

