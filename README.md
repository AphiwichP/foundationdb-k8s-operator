# FoundationDB Kubernetes Operator — Runbook

> **สำหรับใคร:** SRE/DevOps ที่ต้องการ deploy FoundationDB บน GKE ตั้งแต่ต้นจนสำเร็จ
> **เวลาที่ใช้:** ~2-3 ชั่วโมง (รวม debug)
> **Cluster:** GKE `sre-lab-cluster`, zone `asia-southeast1-c`

---

## สิ่งที่ต้องมีก่อนเริ่ม

- `kubectl` configured ชี้ไปที่ GKE cluster แล้ว
- `gcloud` CLI login แล้ว
- `git clone` หรือ copy folder นี้มาแล้ว

---

## Step 0 — Scale Up Cluster

> **[ภาพ: แสดง GKE console ก่อน scale / หลัง scale]**

```bash
gcloud container clusters resize sre-lab-cluster \
  --node-pool=default-pool \
  --num-nodes=3 \
  --zone=asia-southeast1-c

gcloud container clusters get-credentials sre-lab-cluster \
  --zone=asia-southeast1-c \
  --project=sre-lab-493007

kubectl get nodes
```

ต้องเห็น **3 nodes** สถานะ `Ready` ก่อนถึงไปขั้นถัดไปได้

> ทำไมต้อง 3 nodes: FDB `redundancyMode: double` ต้องการ node หลายตัวเพื่อ distribute data และ test fault tolerance ได้จริง

---

## Step 1 — สร้าง Namespace

```bash
kubectl create namespace fdb-operator
```

---

## Step 2 — Apply CRDs (Custom Resource Definitions)

CRD คือการบอก Kubernetes ว่า "มี resource type ใหม่ 3 ตัวนะ"

```bash
kubectl apply -f https://raw.githubusercontent.com/FoundationDB/fdb-kubernetes-operator/main/config/crd/bases/apps.foundationdb.org_foundationdbclusters.yaml

kubectl apply -f https://raw.githubusercontent.com/FoundationDB/fdb-kubernetes-operator/main/config/crd/bases/apps.foundationdb.org_foundationdbbackups.yaml

kubectl apply -f https://raw.githubusercontent.com/FoundationDB/fdb-kubernetes-operator/main/config/crd/bases/apps.foundationdb.org_foundationdbrestores.yaml
```

ตรวจว่า CRDs ลง register แล้ว:

```bash
kubectl get crd | grep foundationdb
```

ต้องเห็น 3 บรรทัด: `foundationdbclusters`, `foundationdbbackups`, `foundationdbrestores`

> **[ภาพ: output ของ kubectl get crd | grep foundationdb]**

---

## Step 3 — Deploy Operator Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/foundationdb/fdb-kubernetes-operator/main/config/samples/deployment.yaml -n fdb-operator
```

รอ operator pod Running:

```bash
kubectl get pods -n fdb-operator -w
```

รอจนเห็น `fdb-kubernetes-operator-controller-manager-xxx` สถานะ `1/1 Running`

> **[ภาพ: kubectl get pods -n fdb-operator แสดง operator Running]**

---

## Step 4 — สร้าง ServiceAccount สำหรับ FDB Pods

> **สำคัญมาก — ถ้าข้ามขั้นตอนนี้ pods จะไม่สามารถ set locality annotations ได้ และ cluster จะ reconcile ไม่สำเร็จ**

```bash
kubectl apply -n fdb-operator -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fdb-kubernetes
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: fdb-kubernetes
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "update", "patch", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: fdb-kubernetes
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: fdb-kubernetes
subjects:
- kind: ServiceAccount
  name: fdb-kubernetes
EOF
```

---

## Step 5 — Deploy FoundationDBCluster CR

Apply ไฟล์ `fdb-cluster.yaml` ที่อยู่ใน repo นี้:

```bash
kubectl apply -f fdb-cluster.yaml
```

รอ cluster reconcile สำเร็จ (อาจใช้เวลา 5-10 นาที):

```bash
kubectl get foundationdbcluster fdb-cluster -n fdb-operator -w
```

รอจนเห็น:
- `RECONCILED` = `GENERATION` (เลขเท่ากัน)
- `AVAILABLE` = `true`
- `FULLREPLICATION` = `true`

> **[ภาพ: kubectl get foundationdbcluster แสดง RECONCILED=3, AVAILABLE=true, FULLREPLICATION=true]**

ตรวจสถานะ cluster ด้วย fdbcli:

```bash
kubectl exec -n fdb-operator fdb-cluster-storage-<pod-id> -c foundationdb -- \
  /usr/bin/fdbcli -C /var/dynamic-conf/fdb.cluster --exec "status" 2>&1 | head -25
```

ต้องเห็น `Replication health - Healthy` และ `Fault Tolerance - 1 machines`

---

## Step 6 — Deploy MinIO (Object Storage สำหรับ Backup)

```bash
kubectl apply -n fdb-operator -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
spec:
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio:latest
        args: ["server", "/data", "--console-address", ":9001"]
        env:
        - name: MINIO_ROOT_USER
          value: "minioadmin"
        - name: MINIO_ROOT_PASSWORD
          value: "minioadmin"
        ports:
        - containerPort: 9000
        - containerPort: 9001
---
apiVersion: v1
kind: Service
metadata:
  name: minio
spec:
  selector:
    app: minio
  ports:
  - name: api
    port: 9000
    targetPort: 9000
  - name: console
    port: 9001
    targetPort: 9001
EOF
```

รอ MinIO Ready แล้วสร้าง bucket:

```bash
kubectl wait --for=condition=ready pod -l app=minio -n fdb-operator --timeout=60s

kubectl run minio-setup --rm -it --restart=Never \
  --image=minio/mc:latest \
  -n fdb-operator \
  --command -- mc alias set minio http://minio:9000 minioadmin minioadmin

kubectl run minio-mb --rm -it --restart=Never \
  --image=minio/mc:latest \
  -n fdb-operator \
  --command -- mc mb minio/fdb-backup --insecure
```

---

## Step 7 — Deploy FoundationDBBackup CR

Apply ไฟล์ `fdb-backup.yaml`:

```bash
kubectl apply -f fdb-backup.yaml
```

รอ backup reconcile สำเร็จ:

```bash
kubectl get foundationdbbackup fdb-backup -n fdb-operator -w
```

รอจนเห็น `RECONCILED=1` และ `RESTORABLE=true`

> **[ภาพ: kubectl get foundationdbbackup แสดง RECONCILED=1, RESTORABLE=true]**

ตรวจ backup agents running:

```bash
kubectl get pods -n fdb-operator | grep backup
```

ต้องเห็น 2 pods สถานะ `1/1 Running`

---

## Step 8 — ทดสอบ Fault Tolerance

**Terminal 1 — Watch pods:**
```bash
kubectl get pods -n fdb-operator -w | grep storage
```

**Terminal 2 — Kill storage pod:**
```bash
kubectl delete pod fdb-cluster-storage-<pod-id> -n fdb-operator
```

> **[ภาพ: terminal แสดง pod Terminating → Pending → Running ภายใน 20-30 วินาที]**

ผลที่คาดหวัง: pod ใหม่ขึ้นมา Running ใน **< 60 วินาที** (lab นี้ได้ ~20 วินาที)

---

## Step 9 — Benchmark Baseline

```bash
# Write benchmark (1000 sequential writes)
kubectl exec -n fdb-operator fdb-cluster-storage-<pod-id> -c foundationdb -- bash -c '
CMDS="writemode on;"
for i in $(seq 1 1000); do
  CMDS="$CMDS set bench_key_$(printf "%04d" $i) bench_val_$(printf "%04d" $i);"
done
START=$(date +%s%N)
/usr/bin/fdbcli -C /var/dynamic-conf/fdb.cluster --exec "$CMDS" > /dev/null 2>&1
END=$(date +%s%N)
ELAPSED=$(( (END - START) / 1000000 ))
echo "1000 writes: ${ELAPSED}ms"
echo "Throughput: $(( 1000000 / ELAPSED )) ops/sec"
'
```

```bash
# Read/status baseline
kubectl exec -n fdb-operator fdb-cluster-storage-<pod-id> -c foundationdb -- \
  /usr/bin/fdbcli -C /var/dynamic-conf/fdb.cluster --exec "status json" 2>&1 | \
  python3 -c "
import json,sys
d=json.load(sys.stdin)
w=d['cluster']['workload']
print('Read requests:', round(w['operations']['read_requests']['hz'], 1), 'ops/s')
print('Keys read:', round(w['keys']['read']['hz'], 1), 'keys/s')
print('Transactions committed:', round(w['transactions']['committed']['hz'], 1), 'tx/s')
"
```

**ผลลัพธ์ที่ได้จาก Lab นี้:**

| Metric | Value |
|---|---|
| Write throughput (sequential single-key tx) | ~20 ops/sec |
| Read requests | ~21.7 ops/sec |
| Keys read | ~25.0 keys/s |
| Transactions committed | ~0.2 tx/s |
| Self-heal time | **~20 seconds** |

---

## Troubleshooting ที่เจอ

### 1. Pods Pending — CPU ไม่พอ
**Symptom:** `0/2 nodes available: Insufficient cpu`
**Fix:** Scale cluster เป็น 3 nodes และลด CPU requests ใน `fdb-cluster.yaml` เป็น `50m`

### 2. RECONCILED ค้าง — locality annotations ไม่ถูก set
**Symptom:** `Pod is ineligible to be a coordinator due to missing locality information`
**Root cause:** Pods ใช้ `default` ServiceAccount ที่ไม่มีสิทธิ์ patch annotations
**Fix:** ทำ Step 4 ก่อน apply cluster CR เสมอ

### 3. Backup timeout — `Could not create backup container`
**Root cause:** FDB blobstore ต้องใช้ `sc=0` เพื่อ skip TLS แม้จะเป็น HTTP
**Fix:** เพิ่ม `sc=0` ใน `urlParameters` และใช้ `--knob_http_verbose_level=3` เพื่อ debug

### 4. `Unknown URL parameter: 'secure'`
**Fix:** ใช้ `sc=0` ไม่ใช่ `secure=0`

---

## ไฟล์ในโปรเจกต์นี้

| ไฟล์ | หน้าที่ |
|---|---|
| `fdb-cluster.yaml` | FoundationDBCluster CR — กำหนด spec ของ FDB cluster |
| `fdb-backup.yaml` | FoundationDBBackup CR — กำหนด backup ไปยัง MinIO |
| `README.md` | Runbook นี้ |
| `runbook-foundationdb-k8s-operator.md` | คำอธิบายเชิงลึกสำหรับทำความเข้าใจ lab |
| `pause-resume.md` | วิธี pause/resume lab เพื่อประหยัด cost |
