# KvRocks Cluster Deployment Guide 🚀

คู่มือสำหรับการ Deploy KvRocks Cluster (2 Master + 2 Replica) บน Kubernetes โดยใช้ Helm Chart

## 📋 สิ่งที่ต้องเตรียม (Prerequisites)

*   **Kubernetes Cluster** (v1.19+)
*   **Helm** (v3.0+) - [วิธีการติดตั้ง Helm](https://helm.sh/docs/intro/install/)
*   **kubectl** ที่ตั้งค่าเชื่อมต่อกับ Cluster เรียบร้อยแล้ว

---

## ⚙️ การตั้งค่าที่สำคัญ (Configuration)

ก่อนทำการติดตั้ง ให้ตรวจสอบค่าในไฟล์ `values.yaml` โดยค่าที่ควรให้ความสำคัญมีดังนี้:

| Parameter | Description | Default |
| :--- | :--- | :--- |
| `auth.password` | รหัสผ่านสำหรับเชื่อมต่อ (จะถูกเก็บเป็น Secret) | `kvrocks-local-2024` |
| `storage.size` | ขนาดพื้นที่เก็บข้อมูลต่อ Node | `1Gi` |
| `storage.storageClassName` | Storage Class ของ Disk (เช่น `standard`, `longhorn`, `gp2`) | `hostpath` |
| `resources` | การจำกัด CPU และ Memory ต่อ Node | `cpu: 500m, mem: 512Mi` |

---

## 🚀 ขั้นตอนการติดตั้ง (Deployment Steps)

### 1. ติดตั้ง Helm Chart
รันคำสั่งด้านล่างเพื่อทำการติดตั้ง Resources ทั้งหมดเข้าสู่ Cluster:

```bash
# ติดตั้งไว้ที่ namespace 'kvrocks' (สร้างใหม่ถ้ายังไม่มี)
helm install kvrocks-cluster ./kvrocks-cluster-deployment -n kvrocks --create-namespace
```

### 2. ตรวจสอบสถานะของ Pod
รอจนกระทั่ง Pod ทั้ง 4 ตัวขึ้นสถานะ `Running` ทั้งหมด:

```bash
kubectl get pods -n kvrocks
```

### 3. เริ่มต้นระบบ Cluster (Cluster Initialization) ‼️ **สำคัญ**
หลังจาก Pod รันแล้ว ระบบจะยังไม่รู้จักกันในฐานะ Cluster ต้องรันสคริปต์เพื่อตั้งค่า Topology:

```bash
# รันสคริปต์ตั้งค่า Master/Slave และแบ่ง Slot อัตโนมัติ
kubectl exec kvrocks-cluster-0 -n kvrocks -- /bin/sh /var/run/kvrocks-init-script/init.sh
```

---

## 🧪 การทดสอบการเข้าใช้งาน (Verification)

### ทดสอบการส่งคำสั่ง PING
```bash
kubectl exec kvrocks-cluster-0 -n kvrocks -- redis-cli -a <รหัสผ่านของคุณ> --no-auth-warning PING
# ผลลัพธ์ที่ควรได้: PONG
```

### ตรวจสอบสถานะ Cluster
```bash
kubectl exec kvrocks-cluster-0 -n kvrocks -- redis-cli -a <รหัสผ่านของคุณ> --no-auth-warning CLUSTER INFO
# ควรเห็น cluster_state: ok และ cluster_known_nodes: 4
```

---

## 🛠 คำสั่งที่มีประโยชน์

*   **อัปเดตการตั้งค่า (Update):** เมื่อแก้ไข `values.yaml` ให้รัน:
    ```bash
    helm upgrade kvrocks-cluster ./kvrocks-cluster-deployment -n kvrocks
    ```
*   **ถอนการติดตั้ง (Uninstall):**
    ```bash
    helm uninstall kvrocks-cluster -n kvrocks
    ```
*   **ดู Logs ของ Node:**
    ```bash
    kubectl logs -f kvrocks-cluster-0 -n kvrocks
    ```

---

**หมายเหตุ:** ไฟล์คอนฟิกจะถูกบันทึกไว้ที่ `/var/lib/kvrocks/kvrocks.conf` ภายใน Pod ซี่งจะมีการเก็บ Node ID ไว้อย่างถาวรแม้มีการ Restart Pod
