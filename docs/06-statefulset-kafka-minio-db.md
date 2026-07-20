# StatefulSet, Kafka, MinIO và Database trong Kubernetes: Quản lý ứng dụng Stateful như thế nào?

Ở các bài trước, ta đã học:

* **Pod, Deployment, ReplicaSet**: cách Kubernetes quản lý workload stateless.
* **Service, DNS, Namespace**: cách các ứng dụng trong cluster tìm thấy nhau.
* **ConfigMap, Secret**: cách inject cấu hình và thông tin nhạy cảm vào ứng dụng.
* **Resource Requests, Limits, Quota**: cách Kubernetes quản lý tài nguyên CPU, Memory và giới hạn sử dụng.

Tuy nhiên, tất cả các ví dụ trước chủ yếu xoay quanh **stateless application**.

Ví dụ:

```text
Frontend
Backend API
REST service
Web server
Microservice
```

Các ứng dụng này có một đặc điểm:

```text
Pod chết -> tạo Pod mới
Không cần giữ dữ liệu cũ
Không quan tâm Pod nào đang chạy
```

Ví dụ:

```text
backend-pod-abc123

DELETE

backend-pod-xyz789
```

Pod mới hoàn toàn có thể thay thế Pod cũ.

---

Nhưng trong thực tế, rất nhiều hệ thống quan trọng lại là **stateful application**:

```text
Database
Kafka
Elasticsearch
Redis cluster
MinIO distributed storage
Zookeeper
Cassandra
RabbitMQ cluster
```

Các hệ thống này có trạng thái cần được giữ:

```text
Data
Identity
Storage
Cluster membership
Network identity
Replication state
```

Ví dụ:

Một PostgreSQL database:

```text
postgres-0

data:
customer table
order table
transaction history
```

Nếu Pod chết và Kubernetes tạo Pod mới:

```text
postgres-new
```

thì:

```text
Database mất dữ liệu
Không biết đây là node nào
Không join được cluster
```

Vì vậy Deployment không phù hợp để chạy các ứng dụng stateful.

Kubernetes cung cấp một workload controller khác:

# StatefulSet

---

# 1. Stateless vs Stateful

Trước khi học StatefulSet, cần hiểu sự khác biệt giữa stateless và stateful.

---

# 1.1 Stateless Application

Stateless nghĩa là:

```text
Application không lưu trạng thái bên trong Pod
```

Ví dụ:

```text
API server
Web server
Frontend
Authentication service
```

Ví dụ có 3 backend Pod:

```
backend-1
backend-2
backend-3
```

Request nào cũng có thể đi vào bất kỳ Pod nào:

```
User
 |
 v
Service
 |
 +--> backend-1
 +--> backend-2
 +--> backend-3
```

Nếu:

```
backend-2 chết
```

Kubernetes tạo:

```
backend-4
```

Không ảnh hưởng.

Vì:

```
Pod chỉ chạy code
Data nằm nơi khác
```

Deployment phù hợp cho mô hình này.

---

# 1.2 Stateful Application

Stateful nghĩa là:

```text
Mỗi instance có identity và data riêng
```

Ví dụ Kafka:

```
kafka-0
kafka-1
kafka-2
```

Mỗi broker có:

```
broker id
log segment
partition replica
metadata
```

Nếu đổi:

```
kafka-0
```

thành:

```
kafka-new
```

thì cluster Kafka có thể không nhận ra.

---

Database cũng tương tự:

```
mysql-0
mysql-1
mysql-2
```

Mỗi node có:

```
database files
transaction log
replication position
```

Không thể tạo Pod mới tùy ý.

---

Vì vậy Stateful application cần:

```
Stable hostname
Stable storage
Stable identity
Ordered startup/shutdown
```

Đó chính là nhiệm vụ của StatefulSet.

---

# 2. StatefulSet là gì?

**StatefulSet là Kubernetes controller dùng để quản lý các workload cần identity ổn định và persistent storage.**

Nói đơn giản:

```
Deployment:
Pod nào cũng giống nhau

StatefulSet:
Mỗi Pod có danh tính riêng
```

Ví dụ Deployment:

```
backend-6f7d8f9-x12ab
backend-6f7d8f9-y34cd
backend-6f7d8f9-z56ef
```

Tên random.

---

StatefulSet:

```
mysql-0
mysql-1
mysql-2
```

Tên cố định.

---

# 3. Đặc điểm quan trọng của StatefulSet

StatefulSet cung cấp 4 tính năng quan trọng:

---

# 3.1 Stable Network Identity

Mỗi Pod có hostname cố định.

Ví dụ:

StatefulSet:

```yaml
replicas: 3
```

Tạo:

```
kafka-0
kafka-1
kafka-2
```

Nếu:

```
kafka-1
```

bị restart:

Trước:

```
kafka-1
IP: 10.244.1.20
```

Sau:

```
kafka-1
IP: 10.244.2.30
```

IP thay đổi.

Nhưng hostname vẫn:

```
kafka-1
```

Ứng dụng biết:

```
Đây vẫn là broker số 1
```

---

# 3.2 Stable Storage

Mỗi Pod có volume riêng.

Ví dụ:

```
mysql-0
 |
 PVC mysql-data-mysql-0


mysql-1
 |
 PVC mysql-data-mysql-1


mysql-2
 |
 PVC mysql-data-mysql-2
```

Nếu Pod bị xóa:

```
mysql-1 DELETE
```

PVC vẫn còn:

```
mysql-data-mysql-1
```

Pod mới mount lại:

```
mysql-1
+
mysql-data-mysql-1
```

Dữ liệu được giữ nguyên.

---

# 3.3 Ordered Deployment

Deployment tạo Pod song song:

```
backend-1
backend-2
backend-3
```

gần như cùng lúc.

---

StatefulSet tạo tuần tự:

```
mysql-0
 |
 Ready

mysql-1
 |
 Ready

mysql-2
 |
 Ready
```

Điều này rất quan trọng với cluster database.

Ví dụ:

```
Leader node phải chạy trước follower node
```

---

# 3.4 Ordered Scaling

Scale StatefulSet:

```bash
kubectl scale statefulset kafka --replicas=5
```

Kubernetes tạo:

```
kafka-3
kafka-4
```

theo thứ tự.

Scale down:

```
kafka-4
kafka-3
```

bị xóa trước.

Không xóa random.

---

# 4. StatefulSet Architecture

Một StatefulSet thường đi cùng:

```
StatefulSet
    |
    |
Headless Service
    |
    |
PersistentVolumeClaim
    |
    |
PersistentVolume
    |
    |
StorageClass
```

Mô hình:

```
             Headless Service

                    |
        +-----------+-----------+
        |           |           |

     kafka-0     kafka-1     kafka-2

        |           |           |

       PVC         PVC         PVC

        |           |           |

       PV          PV          PV
```

---

# 5. Headless Service và StatefulSet

StatefulSet gần như luôn dùng Headless Service.

Ví dụ:

```yaml
clusterIP: None
```

Tại sao?

Vì Stateful application cần biết từng node.

---

Service thường:

```
mysql.default.svc.cluster.local

        |
        |
    ClusterIP

        |
   +----+----+
   |    |    |
mysql0 mysql1 mysql2
```

Client không biết node nào.

---

Nhưng database cluster cần:

```
mysql-0
mysql-1
mysql-2
```

DNS:

```
mysql-0.mysql.default.svc.cluster.local

mysql-1.mysql.default.svc.cluster.local

mysql-2.mysql.default.svc.cluster.local
```

Mỗi node có địa chỉ riêng.

---

# 6. StatefulSet Manifest mẫu

Ví dụ đơn giản:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql
  replicas: 3

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
      - name: mysql
        image: mysql:8
        ports:
        - containerPort: 3306

        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql

  volumeClaimTemplates:

  - metadata:
      name: mysql-data

    spec:
      accessModes:
      - ReadWriteOnce

      resources:
        requests:
          storage: 10Gi
```

---

Khi tạo:

```bash
kubectl apply -f mysql.yaml
```

Kubernetes tạo:

```
mysql-0
mysql-1
mysql-2
```

và PVC:

```
mysql-data-mysql-0

mysql-data-mysql-1

mysql-data-mysql-2
```

---

# 7. PersistentVolume trong StatefulSet

StatefulSet không tự lưu data.

Nó cần:

```
PVC
 |
 PV
 |
 Storage
```

---

Luồng:

```
Application

     |
     v

Pod

     |
     v

PVC

     |
     v

PV

     |
     v

Disk thật
```

---

Ví dụ:

Database ghi:

```
customer.db
transaction.log
```

thực tế nằm ở:

```
Persistent Volume
```

không nằm trong container filesystem.

---

# 8. StorageClass trong StatefulSet

Trong production, thường không tạo PV thủ công.

Ta dùng StorageClass.

Ví dụ:

```yaml
storageClassName: gp3
```

Kubernetes tự tạo volume.

Ví dụ AWS:

```
PVC

 |
 v

StorageClass gp3

 |
 v

AWS EBS Volume
```

---

Ví dụ:

```yaml
volumeClaimTemplates:

- metadata:
    name: data

  spec:

    storageClassName: gp3

    resources:
      requests:
        storage: 100Gi
```

Kubernetes tự:

```
Create EBS volume
Attach volume
Mount vào Pod
```

---

# 9. Kafka chạy trên Kubernetes

Kafka là ví dụ kinh điển của StatefulSet.

Kafka cần:

```
Broker identity
Persistent log
Stable network
Cluster discovery
```

---

Ví dụ Kafka cluster:

```
kafka-0
kafka-1
kafka-2
```

Mỗi broker có:

```
broker.id

partition

log directory
```

---

Architecture:

```
Producer

    |
    v

Kafka Service

    |
    +----------+
    |          |
kafka-0    kafka-1    kafka-2

    |          |          |

 PVC        PVC        PVC
```

---

# 10. Vì sao Kafka không dùng Deployment?

Giả sử Deployment:

```
kafka-abc123
kafka-def456
kafka-xyz789
```

Problem:

Kafka cần biết:

```
Broker 0
Broker 1
Broker 2
```

Nhưng Pod name thay đổi.

Restart:

```
kafka-abc123
        |
        X

new pod:
kafka-qwe999
```

Kafka nghĩ:

```
Đây là broker mới
```

Có thể gây:

```
cluster rebalance
data inconsistency
partition problem
```

---

StatefulSet:

```
kafka-0
kafka-1
kafka-2
```

luôn giữ identity.

---

# 11. MinIO chạy trên Kubernetes

MinIO là distributed object storage.

Nó giống:

```
S3 private
```

Dùng để lưu:

```
Image
Dataset
Backup
ML artifact
Data lake
```

---

Distributed MinIO:

```
minio-0
minio-1
minio-2
minio-3
```

Mỗi node có disk riêng:

```
minio-0
 |
 PVC

minio-1
 |
 PVC

minio-2
 |
 PVC

minio-3
 |
 PVC
```

---

MinIO cần biết:

```
node nào
disk nào
endpoint nào
```

Vì vậy StatefulSet phù hợp.

---

DNS:

```
minio-0.minio.default.svc.cluster.local

minio-1.minio.default.svc.cluster.local
```

MinIO dùng các endpoint này để form cluster.

---

# 12. Database chạy trên Kubernetes

Các database phổ biến:

```
PostgreSQL
MySQL
MongoDB
Redis
Cassandra
ElasticSearch
```

đều có thể chạy bằng StatefulSet.

---

Ví dụ PostgreSQL:

```
postgres-0
postgres-1
postgres-2
```

Có thể:

```
postgres-0 = primary

postgres-1 = replica

postgres-2 = replica
```

---

Storage:

```
postgres-0
 |
 postgres-data-0


postgres-1
 |
 postgres-data-1
```

Nếu:

```
postgres-1 restart
```

nó mount lại:

```
postgres-data-1
```

---

# 13. StatefulSet vs Deployment

|          | Deployment            | StatefulSet      |
| -------- | --------------------- | ---------------- |
| Workload | Stateless             | Stateful         |
| Pod name | Random                | Stable           |
| Identity | Không có              | Có               |
| Storage  | Shared hoặc không cần | PVC riêng        |
| Startup  | Song song             | Theo thứ tự      |
| Scale    | Random                | Ordered          |
| DNS      | Service chung         | DNS từng Pod     |
| Ví dụ    | API, frontend         | Kafka, DB, MinIO |

---

# 14. Khi nào không nên dùng StatefulSet?

Không phải app nào cũng cần StatefulSet.

Không nên dùng cho:

```
REST API
Frontend
Web server
Worker không lưu state
Batch job
```

Ví dụ:

Backend API:

```
backend-1
backend-2
backend-3
```

Deployment tốt hơn.

---

# 15. Troubleshooting StatefulSet

## 15.1 Pod StatefulSet stuck Pending

Kiểm tra:

```bash
kubectl describe pod mysql-0
```

Thường do:

```
PVC chưa bind
StorageClass lỗi
Không đủ storage
Node không phù hợp
```

---

## 15.2 PVC Pending

Kiểm tra:

```bash
kubectl get pvc
```

Ví dụ:

```
mysql-data-mysql-0   Pending
```

Nguyên nhân:

```
Không có StorageClass
Storage provider lỗi
Không tạo được PV
```

---

## 15.3 Pod không start theo thứ tự

Kiểm tra:

```bash
kubectl get pods
```

Ví dụ:

```
mysql-0 Running

mysql-1 Pending
```

Có thể:

```
mysql-0 chưa Ready
database init lỗi
volume lỗi
```

---

## 15.4 Mất dữ liệu sau restart

Kiểm tra:

```bash
kubectl get pvc
```

Nếu dùng:

```
emptyDir
```

thì data mất.

Database phải dùng:

```
PersistentVolumeClaim
```

---

# 16. Bài lab tự làm

## Lab 1: Tạo StatefulSet nginx

Yêu cầu:

```
StatefulSet:
web

replicas:
3
```

Quan sát:

```bash
kubectl get pods
```

Kết quả:

```
web-0
web-1
web-2
```

---

## Lab 2: Scale StatefulSet

```bash
kubectl scale statefulset web --replicas=5
```

Quan sát:

```
web-3
web-4
```

được tạo thêm.

---

## Lab 3: Delete Pod

Xóa:

```bash
kubectl delete pod web-1
```

Quan sát:

```bash
kubectl get pods
```

Pod mới:

```
web-1
```

được tạo lại.

---

## Lab 4: Test PVC

Tạo StatefulSet có:

```
volumeClaimTemplates
```

Kiểm tra:

```bash
kubectl get pvc
```

Kết quả:

```
data-web-0
data-web-1
data-web-2
```

---

# 17. Checklist cần nắm

Sau bài này cần hiểu:

```
Stateful khác Stateless như thế nào?

Tại sao Deployment không phù hợp cho database?

StatefulSet giải quyết vấn đề gì?

Stable hostname là gì?

Stable storage là gì?

volumeClaimTemplates dùng để làm gì?

Headless Service liên quan StatefulSet thế nào?

DNS của Stateful Pod có dạng gì?

Kafka cần StatefulSet vì sao?

MinIO cần StatefulSet vì sao?

Database chạy Kubernetes cần lưu ý gì?

PVC, PV, StorageClass liên quan thế nào?
```

---

# 18. Tổng kết

Kubernetes có hai nhóm workload chính:

## Stateless

Dùng:

```
Deployment
ReplicaSet
Service
```

Ví dụ:

```
Frontend
Backend API
Microservice
```

Pod có thể thay đổi tùy ý.

---

## Stateful

Dùng:

```
StatefulSet
Headless Service
PVC
StorageClass
```

Ví dụ:

```
Kafka
MinIO
PostgreSQL
MySQL
MongoDB
ElasticSearch
```

Mỗi Pod có:

```
Identity riêng
DNS riêng
Storage riêng
Lifecycle riêng
```

Mô hình tổng quát:

```
             Headless Service

                    |
        +-----------+-----------+
        |           |           |

      app-0       app-1       app-2

        |           |           |

       PVC         PVC         PVC

        |           |           |

       PV          PV          PV

        |           |           |

     Storage    Storage    Storage
```

Hiểu chắc StatefulSet là bước quan trọng trước khi đi vào các hệ thống Kubernetes production thực tế như:

```
Kafka cluster
Spark cluster
Data platform
Database HA
ML infrastructure
Cloud-native storage
```

Đây là phần mà Kubernetes bắt đầu từ việc "chạy container" chuyển sang "quản lý cả hệ thống phân tán".
