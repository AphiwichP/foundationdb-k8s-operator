# Context — FoundationDB k8s Operator Lab

## ใครทำงานนี้

- **User:** aphiwich — intern SRE/DevOps
- **Style:** จับมือสอน (hands-on guided) — อธิบายทุก command ก่อนรัน ว่าทำไมถึงทำแบบนี้
- **ภาษา:** คุยภาษาไทยได้เลยครับ

---

## GKE Cluster (shared กับทุก lab)

```
PROJECT_ID   : sre-lab-493007
CLUSTER_NAME : sre-lab-cluster
ZONE         : asia-southeast1-c   ← ต้องใช้ --zone ไม่ใช่ --region (ไม่งั้น 404)
NODE_POOL    : default-pool
NUM_NODES    : 2 (ตอน running) / 0 (ตอน paused)
```

### Scale up cluster ก่อนเริ่ม

```bash
gcloud container clusters resize sre-lab-cluster \
  --node-pool=default-pool \
  --num-nodes=2 \
  --zone=asia-southeast1-c

gcloud container clusters get-credentials sre-lab-cluster \
  --zone=asia-southeast1-c \
  --project=sre-lab-493007

kubectl get nodes  # รอจนเห็น Ready
```

---

## Services ที่รันอยู่แล้วใน Cluster

| Namespace      | Service                      | Status   | หมายเหตุ                           |
| -------------- | ---------------------------- | -------- | ---------------------------------- |
| `zenml-server` | MySQL (mysql:8.0)            | Running  | PVC `data-mysql-0` (8Gi) ยังอยู่  |
| `zenml-server` | ZenML Server (helm managed)  | Running  | Helm release: `zenml-server`       |
| `surrealdb`    | SurrealDB + TiKV HA backend  | Stopped  | Lab ก่อนหน้า — uninstall แล้ว     |

**หมายเหตุ ZenML:** ถ้า node ดับแล้วกลับมา ZenML pods จะขึ้นเองอัตโนมัติ (Helm managed)

### Namespace ที่มีอยู่แล้ว

```bash
kubectl get namespaces
# zenml-server   ← ZenML lab
# surrealdb      ← SurrealDB lab (pods ถูก uninstall แล้ว แต่ namespace ยังอยู่)
# kube-system    ← system
```

---

## Lab ที่ทำมาแล้ว (context สำหรับเข้าใจพื้นฐาน)

### Lab ก่อนหน้า: Custom SurrealDB Helm Chart (เสร็จแล้ว ✅)

โค้ดอยู่ที่ `~/lodash-work/custom-surrealdb-chart/`

สิ่งที่ทำ:
- สร้าง Helm chart จาก scratch สำหรับ SurrealDB + TiKV HA backend
- StatefulSet สำหรับ SurrealDB (1 replica) + PD (3 replicas) + TiKV (3 replicas)
- จัดการ auth ผ่าน Kubernetes Secret
- Schema init ผ่าน ConfigMap
- Publish chart ไป GCP Artifact Registry

Chart อยู่ที่:
```
oci://asia-southeast1-docker.pkg.dev/sre-lab-493007/helm-charts/surrealdb-custom:0.1.0
```

### ทักษะที่ได้จาก Lab ก่อนหน้า

- Helm chart structure: `_helpers.tpl`, `values.yaml`, `templates/`
- StatefulSet vs Deployment: ทำไม stateful workloads ต้องใช้ StatefulSet
- Headless Service: ทำไม distributed systems ต้องการ DNS per-pod
- Kubernetes Operator concept: TiKV ใช้ PD เป็น "brain" — คล้ายกับ pattern ของ Operator
- Debug pods บน cluster จริง: OOMKill, CrashLoopBackOff, rolling update stuck

---

## Lab นี้: FoundationDB k8s Operator

FoundationDB เป็น distributed database ของ Apple ที่เน้น:
- ACID transactions ที่ scale ได้
- Multi-model (ใช้ layer API ด้านบน)
- Proven in production: iCloud, Snowflake ใช้

**ทำไม Operator ไม่ใช่ Helm chart แบบ lab ก่อน?**

Helm chart เหมาะกับ stateless/simple stateful apps
FoundationDB มี lifecycle ที่ซับซ้อนกว่า (cluster membership, recovery, backup) ที่ต้องการ
custom controller ที่รู้จัก FoundationDB โดยเฉพาะ → นั่นคือ Kubernetes Operator

**Operator = Custom Controller + CRD**
- CRD (Custom Resource Definition): บอก k8s ว่ามี resource type ใหม่ชื่อ `FoundationDBCluster`
- Controller: loop ที่ watch CRD แล้วทำให้ actual state = desired state

---

## Artifact Registry ที่มีอยู่แล้ว

```
Registry: asia-southeast1-docker.pkg.dev/sre-lab-493007/helm-charts
Format:   Docker/OCI

เข้าถึงด้วย:
helm registry login asia-southeast1-docker.pkg.dev \
  --username oauth2accesstoken \
  --password "$(gcloud auth print-access-token)"
```

---

## Pause/Resume Reference

ดูไฟล์: `~/lodash-work/custom-surrealdb-chart/pause-resume.md`

สำหรับ lab นี้ให้สร้าง `pause-resume.md` แยกใน folder นี้เมื่อ deploy เสร็จ

---

## ก่อนเริ่ม task

```bash
# 1. ตรวจ cluster พร้อมไหม
kubectl get nodes

# 2. ตรวจ namespace ที่มีอยู่
kubectl get namespaces

# 3. ตรวจไม่ให้ชน ZenML
kubectl get pods -n zenml-server
```
