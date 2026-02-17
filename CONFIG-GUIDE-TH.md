# คู่มือการตั้งค่า KvRocks (Configuration Guide)

คู่มือนี้อธิบายเกี่ยวกับตัวแปรต่างๆ ใน `kvrocks.conf` (จาก `templates/configmap.yaml`), วิธีการนำไปใช้งาน (Apply), และบทบาทในขั้นตอนการเริ่มต้นระบบ Cluster (Initialization)

## 1. รายละเอียดตัวแปร (`kvrocks.conf`)

| ตัวแปร | คำอธิบาย | แหล่งที่มาใน `values.yaml` |
| :--- | :--- | :--- |
| `bind` | อินเตอร์เฟซเครือข่ายที่จะเปิดรับการเชื่อมต่อ โดย `0.0.0.0` จะยอมรับการเชื่อมต่อจากทุกที่ภายใน Cluster | `kvrocksConfig.bind` |
| `port` | พอร์ตสำหรับโปรโตคอล Redis ค่าเริ่มต้นคือ `6379` | `kvrocksConfig.port` |
| `cluster-enabled` | เปิดใช้งานโหมด Cluster จำเป็นต้องตั้งค่าเป็น `yes` เพื่อให้คำสั่ง Cluster (`CLUSTERX`) ทำงานได้ | `kvrocksConfig.cluster_enabled` |
| `requirepass` | รหัสผ่านสำหรับการยืนยันตัวตนของ Client | `auth.password` (ผ่าน Secret) |
| `masterauth` | รหัสผ่านที่ใช้โดย Slave เพื่อทำการดึงข้อมูล (Replicate) จาก Master | `auth.password` (ผ่าน Secret) |
| `dir` | ไดเรกทอรีสำหรับเก็บข้อมูลและไฟล์ Log | `kvrocksConfig.dir` |
| `rocksdb.*` | พารามิเตอร์สำหรับการปรับแต่ง (Tuning) ของ RocksDB storage engine (เช่น memory buffers, file sizes, background threads) | `kvrocksConfig.rocksdb_*` |

---

## 2. การจัดการความปลอดภัยและรหัสผ่าน (External Secrets)

สำหรับระดับ Production ที่มีการใช้ **HashiCorp Vault** หรือ **Sealed Secrets**:
ระบบจะไม่สร้าง Secret ใหม่ แต่จะไปดึงมาจากที่ที่มีอยู่แล้ว โดยตั้งค่าดังนี้:

```yaml
# ตัวอย่างการตั้งค่าใน values.yaml เพื่อใช้ Secret จากภายนอก (เช่น Vault)
auth:
  useExistingSecret: true    # เปิดใช้งาน Secret ที่มีอยู่แล้ว
  existingSecret: "my-vault-secret" # ชื่อ Secret ที่ External Secrets Operator สร้างขึ้น
  secretKey: "kv-password"   # Key ภายใน Secret นั้นที่เก็บรหัสผ่าน
```

---

## 3. รายละเอียดตัวแปรทั้งหมดใน `values.yaml`

| Group | Parameter | Description | Why we need it? |
| :--- | :--- | :--- | :--- |
| **Global** | `replicaCount` | จำนวน Pod ทั้งหมด | กำหนดขนาดของ Cluster (ควรสัมพันธ์กับ Master/Replica) |
| **Image** | `image.tag` | Version ของ KvRocks | ใช้ระบุรุ่นที่ต้องการ (แนะนำระบุเป็นเลขแทน `latest`) |
| **Auth** | `auth.useExistingSecret` | สลับโหมดการเก็บรหัสผ่าน | `false`: สร้าง Secret ใหม่ / `true`: ใช้ของเดิม (เช่นจาก Vault) |
| | `auth.password` | รหัสผ่าน (ถ้าสร้างใหม่) | ใช้ล็อกอินและทำ Replication ระหว่างกัน |
| **Resources**| `resources.limits.memory`| จำกัด RAM ต่อ Pod | ป้องกัน Pod กิน RAM จนเครื่อง Host ล้ม (KvRocks ใช้ Mem สำคัญมาก) |
| | `resources.requests.cpu` | จอง CPU ขั้นต่ำ | ประกันความเร็วในการประมวลผลคำสั่ง |
| **Storage** | `storage.size` | พื้นที่ Disk ต่อ Node | KvRocks เก็บข้อมูลลง Disk (ต่างจาก Redis) จึงต้องกะขนาดให้ดี |
| | `storage.storageClassName`| ประเภทของ Disk | เลือกความเร็ว Disk (เช่น `ssd`, `fast`, `hostpath`) |
| **Cluster** | `cluster.masterCount` | จำนวนเครื่อง Master | จำนวนเครื่องที่สามารถ "เขียน" ข้อมูลได้ (ต้องระบุในสคริปต์ init) |
| **Config** | `kvrocksConfig.dir` | ที่เก็บข้อมูลภายใน Pod | ต้องเป็นโฟลเดอร์เดียวกับที่ Mount Storage ไว้ |
| | `kvrocksConfig.rocksdb_*` | RocksDB Tuning | ปรับแต่ง Buffer และการเขียนลง Disk ให้เหมาะกับ RAM ที่มี (ถ้า RAM น้อยต้องปรับค่าเหล่านี้ลง) |

---

## 4. ขั้นตอนการนำคอนฟิกไปใช้งาน (Application Flow)

---

## 3. ความเกี่ยวขอ้งกับสคริปต์ `init cluster`

สคริปต์ `init cluster` (จาก `templates/cluster-init-script.yaml`) ขึ้นอยู่กับการตั้งค่าเหล่านี้:

- **`cluster-enabled yes`**: หากไม่ได้ตั้งค่านี้ คำสั่ง `CLUSTERX` ที่ใช้ใน `init.sh` จะล้มเหลว
- **`requirepass`**: สคริปต์ `init.sh` ใช้คำสั่ง `redis-cli -a "$PASSWORD"` เพื่อเชื่อมต่อกับ Node ต่างๆ ดังนั้นรหัสผ่านในนี้ต้องตรงกับที่ระบุใน `kvrocks.conf`
- **`port`**: สคริปต์จะเชื่อมต่อกับ Node ต่างๆ ผ่านพอร์ต `6379` (ตามที่กำหนดใน `kvrocks.conf`)

**สรุป**: ไฟล์ `.conf` มีหน้าที่เตรียมความพร้อมให้แต่ละ Node สามารถทำงานในโหมด Cluster ได้ ส่วนสคริปต์ `init.sh` คือตัวที่ทำหน้าที่ "เชื่อมต่อ" Node เหล่านั้นเข้าด้วยกันตามโครงสร้าง (Topology) ที่กำหนด (Masters, Slaves, และ Slots)
