# Volume, PVC và StorageClass trong Kubernetes: Quản lý dữ liệu bền vững cho ứng dụng

Ở các bài trước, chúng ta đã học:

* **Pod** là nơi container chạy.
* **Deployment** quản lý vòng đời Pod stateless.
* **Service** giúp các Pod giao tiếp ổn định thông qua DNS.
* **Namespace** giúp phân chia tài nguyên trong cluster.

Tuy nhiên, toàn bộ những thứ trên mới chỉ giải quyết vấn đề:

```text
Làm sao chạy application?
Làm sao scale application?
Làm sao các component giao tiếp với nhau?
```

Còn một vấn đề cực kỳ quan trọng:

```text
Dữ liệu của application nằm ở đâu?
```

Ví dụ:

Một container MySQL đang chạy:

```text
mysql-container
        |
        |
     /var/lib/mysql
        |
        |
    database files
```

Nếu container bị restart thì sao?

Nếu Pod bị xóa thì sao?

Nếu Kubernetes chuyển Pod sang node khác thì sao?

Nếu không có cơ chế lưu trữ:

```text
Pod chết
    |
    v
Container tạo lại
    |
    v
Database mất dữ liệu
```

Đây là lý do Kubernetes cung cấp cơ chế:

```text
Volume
PersistentVolume (PV)
PersistentVolumeClaim (PVC)
StorageClass
```

---

# 1. Vấn đề: Container không lưu dữ liệu

Container về bản chất là ephemeral.

Có nghĩa:

```text
Container tồn tại
    |
    v
Data tồn tại

Container bị xóa
    |
    v
Data mất
```

Ví dụ:

Chạy một container nginx:

```bash
docker run nginx
```

Trong container:

```
/usr/share/nginx/html
```

Ta tạo file:

```bash
echo hello > /usr/share/nginx/html/test.txt
```

File tồn tại.

Nhưng nếu:

```bash
docker rm container
```

thì:

```
test.txt biến mất
```

Kubernetes Pod cũng tương tự.

Ví dụ:

```text
Pod mysql-abc123

/var/lib/mysql
    |
    +-- users.ibd
    +-- orders.ibd
```

Pod crash:

```
mysql-abc123 deleted
```

Pod mới:

```
mysql-def456
```

Kết quả:

```
/var/lib/mysql
(empty)
```

Database mất.

Vì vậy:

> Container filesystem không phù hợp để lưu dữ liệu quan trọng.

---

# 2. Volume trong Kubernetes là gì?

**Volume** là cơ chế mount storage vào container.

Nói đơn giản:

```text
Volume = ổ đĩa được gắn vào container
```

Ví dụ:

Không dùng Volume:

```
Pod

+----------------+
| Container      |
|                |
| /app/data      |
|                |
+----------------+
```

Có Volume:

```
Pod

+----------------+
| Container      |
|                |
| /app/data      |
+-------|--------+
        |
        |
     Volume
        |
        |
 External storage
```

Container chỉ nhìn thấy:

```
/app/data
```

nhưng dữ liệu thật nằm ở Volume.

---

# 3. Volume lifecycle khác Container lifecycle

Đây là điểm rất quan trọng.

Container:

```
short-lived
```

Volume:

```
có thể tồn tại lâu hơn container
```

Ví dụ:

```
Pod A
 |
 | mount
 |
Volume
 |
 |
Database data
```

Pod A bị xóa:

```
Pod A deleted
```

Nhưng:

```
Volume vẫn còn
```

Pod mới:

```
Pod B
 |
 | mount
 |
Volume cũ
```

Data vẫn còn.

---

# 4. Các loại Volume phổ biến

Kubernetes hỗ trợ rất nhiều loại volume:

```text
emptyDir
hostPath
configMap
secret
persistentVolumeClaim
nfs
awsElasticBlockStore
azureDisk
gcePersistentDisk
CSI drivers
```

Trong thực tế cần tập trung:

```text
emptyDir
hostPath
PVC
StorageClass
CSI
```

---

# 5. emptyDir Volume

## 5.1 emptyDir là gì?

`emptyDir` là loại Volume đơn giản nhất.

Kubernetes tạo một thư mục rỗng khi Pod được tạo.

Ví dụ:

```yaml
volumes:
  - name: data
    emptyDir: {}
```

Sau đó mount vào container:

```yaml
containers:
  - name: app
    image: nginx
    volumeMounts:
      - name: data
        mountPath: /data
```

Mô hình:

```
Pod

+----------------+
| Container      |
|                |
| /data          |
|     |          |
+-----|----------+
      |
      |
 emptyDir volume
```

---

## 5.2 Lifecycle của emptyDir

emptyDir tồn tại trong lifecycle của Pod.

Ví dụ:

```
Pod created

      |
      v

emptyDir created

      |
      v

Container restart

      |
      v

Data vẫn còn


Pod deleted

      |
      v

emptyDir deleted
```

Quan trọng:

Container restart:

```
data còn
```

Pod restart:

```
data mất
```

---

## 5.3 Khi nào dùng emptyDir?

Dùng cho dữ liệu tạm:

Ví dụ:

### Temporary cache

```
Application
    |
    |
 /cache
```

### Sharing data giữa containers trong cùng Pod

Ví dụ:

Pod có 2 container:

```
Pod

+-------------+
| App         |
|             |
| write file  |
+-------------+

+-------------+
| Sidecar     |
|             |
| read file   |
+-------------+

        |
        |
    emptyDir
```

Ví dụ:

Nginx + log collector:

```
nginx
 |
 | write log
 |
emptyDir
 |
 |
fluent-bit
 |
 |
send log
```

---

## 5.4 Ví dụ manifest emptyDir

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: cache-volume
          mountPath: /cache

  volumes:
    - name: cache-volume
      emptyDir: {}
```

Kiểm tra:

```bash
kubectl exec -it emptydir-demo -- bash
```

Trong container:

```bash
echo hello > /cache/test.txt
```

Restart container:

```
data còn
```

Delete Pod:

```bash
kubectl delete pod emptydir-demo
```

Data mất.

---

# 6. hostPath Volume

## 6.1 hostPath là gì?

`hostPath` mount trực tiếp filesystem của Kubernetes Node vào Pod.

Ví dụ:

Node:

```
Worker Node

/home/data
     |
     |
     v

Pod container

/app/data
```

Manifest:

```yaml
volumes:
  - name: host-volume
    hostPath:
      path: /home/data
```

---

# 6.2 Ví dụ

Node:

```
worker-node-1

/home/data/file.txt
```

Pod:

```yaml
volumeMounts:
- name: host-volume
  mountPath: /data
```

Trong container:

```
/data/file.txt
```

chính là:

```
worker-node-1:/home/data/file.txt
```

---

# 6.3 Nhược điểm hostPath

hostPath phụ thuộc node.

Ví dụ:

Ban đầu:

```
Pod
 |
 |
worker-node-1
 |
 |
/home/data
```

Kubernetes move Pod:

```
Pod
 |
 |
worker-node-2
```

Nhưng:

```
worker-node-2:/home/data
(empty)
```

Data mất.

---

# 6.4 Khi nào dùng hostPath?

Chỉ nên dùng cho:

```text
Node monitoring
Logging agent
DaemonSet
Testing
```

Ví dụ:

Prometheus node exporter:

```
Node filesystem
        |
        |
   node-exporter Pod
```

Không nên dùng cho:

```
Database
Application data
Production storage
```

---

# 7. PersistentVolume (PV)

## 7.1 Vấn đề của Volume thường

Các volume như:

```
emptyDir
hostPath
```

gắn chặt với Pod hoặc Node.

Production cần:

```
Storage độc lập với Pod
```

Ví dụ:

Database:

```
MySQL Pod

     |
     |
     v

Persistent Storage
```

Pod chết:

```
MySQL Pod deleted
```

Storage vẫn còn.

---

# 7.2 PersistentVolume là gì?

**PersistentVolume (PV)** là storage resource trong Kubernetes cluster.

Nó đại diện cho:

```
một phần storage thật
```

Ví dụ:

```
AWS EBS volume
Azure Disk
NFS
Ceph
SAN
Local SSD
```

Kubernetes nhìn nó như:

```
PV
 |
 |
Storage backend
```

---

# 7.3 PV giống Node như thế nào?

Có thể hiểu:

Node cung cấp CPU/RAM:

```
Node
 |
 |
Pod
```

PV cung cấp Storage:

```
PV
 |
 |
PVC
 |
 |
Pod
```

---

# 8. PersistentVolumeClaim (PVC)

## 8.1 PVC là gì?

PVC là request xin storage của application.

Nếu:

PV = ổ cứng thật

thì:

PVC = đơn xin cấp ổ cứng

Ví dụ developer nói:

```
Tôi cần:

10GB storage
ReadWriteOnce
SSD
```

Tạo PVC:

```yaml
kind: PersistentVolumeClaim
```

Kubernetes tìm PV phù hợp.

---

# 8.2 Quan hệ PV và PVC

Luồng:

```
Admin tạo storage

        |
        v

PersistentVolume

        |
        v

Developer tạo claim

        |
        v

PersistentVolumeClaim

        |
        v

Pod mount PVC

        |
        v

Application sử dụng storage
```

---

# 9. Ví dụ PV + PVC thủ công

## 9.1 Tạo PV

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi

  accessModes:
    - ReadWriteOnce

  hostPath:
    path: /data/mysql
```

---

## 9.2 Tạo PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc

spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 10Gi
```

---

## 9.3 Mount PVC vào Pod

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: mysql

spec:

  containers:
    - name: mysql
      image: mysql:8

      volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql


  volumes:

    - name: mysql-data

      persistentVolumeClaim:
        claimName: mysql-pvc
```

Luồng:

```
MySQL container

/var/lib/mysql

        |
        |
        v

PVC mysql-pvc

        |
        |
        v

PV mysql-pv

        |
        |
        v

/data/mysql
```

---

# 10. Access Mode của Storage

PVC có:

```yaml
accessModes:
```

Có 3 loại chính:

---

## ReadWriteOnce (RWO)

```
Một node có thể mount read/write
```

Ví dụ:

```
Node 1

MySQL Pod
 |
 PVC
```

Phổ biến nhất cho database.

---

## ReadOnlyMany (ROX)

```
Nhiều node đọc
```

Ví dụ:

```
Many Pods
 |
 |
Read only data
```

---

## ReadWriteMany (RWX)

```
Nhiều node cùng đọc ghi
```

Ví dụ:

```
Pod A
Pod B
Pod C

      |
      |
   Shared Storage
```

Thường cần:

```
NFS
CephFS
EFS
Azure Files
```

---

# 11. Volume trong Deployment

Ví dụ backend cần upload file:

```
User upload image

       |
       v

Backend Pod

       |
       v

/mnt/uploads

       |
       v

PVC

       |
       v

Storage
```

Deployment:

```yaml
spec:
  template:

    spec:

      containers:

      - name: backend

        image: backend:v1

        volumeMounts:

        - name: uploads

          mountPath: /app/uploads


      volumes:

      - name: uploads

        persistentVolumeClaim:

          claimName: upload-pvc
```

---

# 12. Vấn đề khi scale Deployment với PVC

Đây là điểm rất quan trọng.

Ví dụ:

Deployment:

```yaml
replicas: 3
```

Pod:

```
backend-1
backend-2
backend-3
```

Mount:

```
same PVC
```

Nếu PVC:

```
ReadWriteOnce
```

thì có thể lỗi:

```
Multi-Attach error
```

Vì:

```
RWO chỉ cho phép mount trên một node
```

Với app stateless:

```
Không nên lưu data local trong Pod.
```

Nên dùng:

```
Object Storage
S3
MinIO
Database riêng
RWX storage
```

---

# 13. Tóm tắt phần 1

Đến đây cần hiểu:

```
Container filesystem
        |
        v
Không bền vững
```

Volume giải quyết:

```
Container
 |
Volume
 |
Storage
```

Các loại:

```
emptyDir
    |
    +--> temporary data

hostPath
    |
    +--> node filesystem

PV/PVC
    |
    +--> persistent storage production
```

Quan hệ:

```
Pod
 |
 | mount
 |
PVC
 |
 |
PV
 |
 |
Storage backend
```

---

Mô hình lúc này:

```text
Pod
 |
 | mount
 |
PVC
 |
 |
PV
 |
 |
Storage backend
```

Tuy nhiên, cách tạo PV thủ công vẫn còn nhiều hạn chế.

Ví dụ:

Một cluster production có:

```text
100 developer
500 application
1000 database instance
```

Nếu mỗi lần có ứng dụng mới, admin phải tự tạo:

```text
PersistentVolume
Storage size
Storage backend
Access mode
```

thì sẽ rất khó quản lý.

Kubernetes giải quyết vấn đề này bằng:

```text
StorageClass
Dynamic Provisioning
CSI Driver
```

---

# 14. StorageClass là gì?

## 14.1 Vấn đề khi tạo PV thủ công

Ở phần trước, chúng ta tạo PV:

```yaml
kind: PersistentVolume

spec:
  capacity:
    storage: 10Gi
```

Nghĩa là admin phải biết:

```text
Storage backend nào?
Dung lượng bao nhiêu?
Loại disk gì?
Replication ra sao?
```

Ví dụ:

AWS:

```text
Create EBS volume
     |
     |
Create PV
     |
     |
Bind PVC
```

Azure:

```text
Create Azure Disk
     |
     |
Create PV
     |
     |
Bind PVC
```

Quá nhiều thao tác thủ công.

---

# 14.2 StorageClass giải quyết vấn đề này

**StorageClass mô tả loại storage mà Kubernetes có thể tạo tự động.**

Nói đơn giản:

```text
StorageClass = template để tạo storage
```

Ví dụ:

```text
StorageClass

fast-ssd

      |
      |
      v

Create SSD volume

      |
      |
      v

PersistentVolume

      |
      |
      v

PVC
```

Developer chỉ cần nói:

```text
Tôi cần 50GB SSD
```

Kubernetes tự tạo:

```text
PV
Storage volume
Mount vào Pod
```

---

# 15. Static Provisioning vs Dynamic Provisioning

Có hai cách tạo storage trong Kubernetes.

---

# 15.1 Static Provisioning

Đây là cách ở phần trước.

Luồng:

```text
Admin

    |
    |
Create PV manually

    |
    |
Developer

    |
    |
Create PVC

    |
    |
Bind
```

Ví dụ:

```text
Admin tạo:

mysql-pv
10Gi

Developer tạo:

mysql-pvc
10Gi
```

Kubernetes tìm PV phù hợp.

Nhược điểm:

```text
- nhiều thao tác
- khó scale
- phụ thuộc admin
- không phù hợp production lớn
```

---

# 15.2 Dynamic Provisioning

Đây là cách production thường dùng.

Luồng:

```text
Developer

     |
     |
Create PVC

     |
     |
StorageClass

     |
     |
Provisioner

     |
     |
Create PV automatically

     |
     |
Attach storage
```

Ví dụ:

PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: mysql-data

spec:

  storageClassName: fast

  accessModes:
    - ReadWriteOnce

  resources:

    requests:

      storage: 20Gi
```

Kubernetes thấy:

```text
storageClassName: fast
```

nó gọi StorageClass `fast`.

Sau đó:

```text
Create disk
Create PV
Bind PVC
```

Developer không cần quan tâm storage thật nằm ở đâu.

---

# 16. StorageClass Manifest

Ví dụ StorageClass trên cloud:

```yaml
apiVersion: storage.k8s.io/v1

kind: StorageClass

metadata:

  name: fast


provisioner:

  kubernetes.io/aws-ebs


parameters:

  type: gp3


reclaimPolicy:

  Delete


volumeBindingMode:

  WaitForFirstConsumer
```

---

Giải thích từng phần:

---

## 16.1 provisioner

```yaml
provisioner:
  kubernetes.io/aws-ebs
```

Cho Kubernetes biết ai chịu trách nhiệm tạo storage.

Ví dụ:

AWS:

```text
aws-ebs
```

Azure:

```text
disk.csi.azure.com
```

GCP:

```text
pd.csi.storage.gke.io
```

NFS:

```text
nfs.csi.k8s.io
```

---

## 16.2 parameters

Ví dụ:

```yaml
parameters:
  type: gp3
```

Cho biết loại storage.

AWS:

```text
gp3
io2
st1
```

Azure:

```text
Premium_LRS
StandardSSD_LRS
```

---

## 16.3 reclaimPolicy

Khi PVC bị xóa, chuyện gì xảy ra với PV?

Có 2 lựa chọn:

---

## Delete

```yaml
reclaimPolicy: Delete
```

Nghĩa là:

```text
PVC deleted

     |
     v

PV deleted

     |
     v

Storage thật bị xóa
```

Phù hợp:

```text
Temporary application
Testing environment
```

---

## Retain

```yaml
reclaimPolicy: Retain
```

Nghĩa là:

```text
PVC deleted

     |
     v

PV giữ lại

     |
     v

Data vẫn còn
```

Phù hợp:

```text
Database production
Important data
```

Ví dụ:

```text
PostgreSQL
MySQL
Kafka
ElasticSearch
```

---

# 17. CSI Driver là gì?

## 17.1 Vấn đề

Kubernetes không tự biết cách tạo:

```text
AWS EBS
Azure Disk
Ceph
NFS
NetApp
```

Mỗi storage provider có API khác nhau.

Ví dụ:

AWS:

```text
CreateVolume()
AttachVolume()
DetachVolume()
DeleteVolume()
```

Azure:

```text
CreateDisk()
AttachDisk()
```

Kubernetes cần một chuẩn chung.

Đó là:

# Container Storage Interface (CSI)

---

# 17.2 CSI Architecture

Mô hình:

```text
             Kubernetes

                  |
                  |
             CSI Interface

                  |
        ---------------------

        |                   |

     AWS CSI            Azure CSI

        |                   |

      EBS                Azure Disk

```

Kubernetes chỉ gọi CSI.

CSI driver chịu trách nhiệm làm việc với storage thật.

---

# 17.3 CSI Driver trong thực tế

Ví dụ:

AWS EKS:

```text
ebs.csi.aws.com
```

Azure AKS:

```text
disk.csi.azure.com
```

Google GKE:

```text
pd.csi.storage.gke.io
```

On-prem:

```text
ceph.csi.ceph.com
nfs.csi.k8s.io
```

---

# 18. Default StorageClass

Trong nhiều cluster, Kubernetes có sẵn một StorageClass mặc định.

Kiểm tra:

```bash
kubectl get storageclass
```

Ví dụ:

```text
NAME             PROVISIONER

standard(default) kubernetes.io/aws-ebs

premium          disk.csi.azure.com
```

Dấu:

```text
(default)
```

nghĩa là:

Nếu PVC không khai báo:

```yaml
storageClassName:
```

Kubernetes dùng StorageClass mặc định.

---

Ví dụ PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim

metadata:
  name: data

spec:

  resources:

    requests:

      storage: 10Gi

  accessModes:

    - ReadWriteOnce
```

Không có:

```yaml
storageClassName
```

Kubernetes tự dùng:

```text
default StorageClass
```

---

# 19. Volume Binding Mode

StorageClass có:

```yaml
volumeBindingMode:
```

Hai giá trị:

```text
Immediate
WaitForFirstConsumer
```

---

# 19.1 Immediate

Mặc định cũ.

Luồng:

```text
Create PVC

    |
    |
Create PV immediately

    |
    |
Pod schedule later
```

Vấn đề:

Storage có thể tạo ở node/zone không phù hợp.

---

# 19.2 WaitForFirstConsumer

Production thường dùng.

Luồng:

```text
Create PVC

        |
        |
Wait

        |
        |
Create Pod

        |
        |
Scheduler chọn node

        |
        |
Create storage phù hợp node
```

Ví dụ AWS:

Cluster:

```text
Zone A
Zone B
Zone C
```

Pod chạy:

```text
Zone B
```

Storage được tạo:

```text
Zone B
```

Tránh lỗi:

```text
Volume node affinity conflict
```

---

# 20. Stateful Application và Storage

Đây là phần rất quan trọng.

Kubernetes ban đầu được thiết kế mạnh cho:

```text
Stateless application
```

Ví dụ:

```text
Frontend
API
Backend
Worker
```

Các app này:

```text
Pod chết -> tạo Pod mới
Không cần giữ identity
```

Nhưng database thì khác.

Ví dụ:

```text
MySQL
Kafka
MongoDB
ElasticSearch
```

Chúng cần:

```text
Data persistence
Stable identity
Stable network
```

---

# 21. StatefulSet và PVC

Deployment:

```text
backend-abc123
backend-def456
```

Pod name thay đổi.

Không phù hợp database.

StatefulSet tạo identity ổn định:

Ví dụ:

```text
mysql-0
mysql-1
mysql-2
```

Mỗi Pod có PVC riêng:

```text
mysql-0

   |
   |
mysql-0-pvc


mysql-1

   |
   |
mysql-1-pvc
```

---

# 22. Ví dụ StatefulSet + VolumeClaimTemplate

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

          storage: 20Gi
```

Kubernetes tạo:

```text
mysql-0

 |
 |
mysql-data-mysql-0


mysql-1

 |
 |
mysql-data-mysql-1


mysql-2

 |
 |
mysql-data-mysql-2
```

Mỗi node database có storage riêng.

---

# 23. Database trên Kubernetes nên lưu data thế nào?

Một câu hỏi phỏng vấn rất hay:

> Có nên chạy database trong Kubernetes không?

Câu trả lời:

Có thể, nhưng cần hiểu storage.

---

## Option 1: Managed Database

Cloud production thường dùng:

```text
AWS RDS
Azure Database
Google Cloud SQL
```

Ưu điểm:

```text
Backup
Replication
Maintenance
Monitoring
```

---

## Option 2: StatefulSet + PVC

Ví dụ:

```text
PostgreSQL
MySQL
MongoDB
Kafka
```

Cần:

```text
Persistent Storage
Backup
Replication
Monitoring
```

---

# 24. Backup và Restore Storage

PVC không thay thế backup.

Ví dụ:

```text
Database

     |
     |
PVC

     |
     |
Disk
```

Nếu:

```text
Human delete database
Corrupt data
Application bug
```

PVC vẫn giữ data lỗi.

Cần backup:

```text
Snapshot
Replication
Database backup
Object storage
```

---

# 25. Volume Snapshot

Kubernetes hỗ trợ snapshot volume.

Ví dụ:

```text
PVC

 |
 |
Snapshot

 |
 |
New PVC
```

Manifest:

```yaml
apiVersion: snapshot.storage.k8s.io/v1

kind: VolumeSnapshot

metadata:

  name: mysql-backup


spec:

  source:

    persistentVolumeClaimName: mysql-pvc
```

Dùng để:

```text
Backup nhanh
Clone database
Test environment
```

---

# 26. Thực hành Lab đầy đủ

## Lab 1: Kiểm tra StorageClass

```bash
kubectl get storageclass
```

Ví dụ:

```text
standard(default)
```

---

# Lab 2: Tạo PVC

File:

```yaml
apiVersion: v1

kind: PersistentVolumeClaim

metadata:

  name: test-pvc


spec:

  accessModes:

  - ReadWriteOnce


  resources:

    requests:

      storage: 1Gi
```

Apply:

```bash
kubectl apply -f pvc.yaml
```

Kiểm tra:

```bash
kubectl get pvc
```

Kết quả:

```text
STATUS Bound
```

---

# Lab 3: Mount PVC vào Pod

```yaml
apiVersion: v1

kind: Pod

metadata:

  name: storage-test


spec:

  containers:

  - name: nginx

    image: nginx


    volumeMounts:

    - name: data

      mountPath: /data


  volumes:

  - name: data

    persistentVolumeClaim:

      claimName: test-pvc
```

Apply:

```bash
kubectl apply -f pod.yaml
```

---

Test:

```bash
kubectl exec -it storage-test -- bash
```

Tạo file:

```bash
echo hello > /data/test.txt
```

Xóa Pod:

```bash
kubectl delete pod storage-test
```

Tạo lại Pod.

Kiểm tra:

```bash
cat /data/test.txt
```

Kết quả:

```text
hello
```

Data vẫn còn.

---

# 27. Troubleshooting PVC

## 27.1 PVC Pending

Kiểm tra:

```bash
kubectl get pvc
```

Ví dụ:

```text
NAME STATUS

data Pending
```

Nguyên nhân:

```text
Không có StorageClass
Không có provisioner
Storage backend lỗi
Request size quá lớn
```

Debug:

```bash
kubectl describe pvc data
```

---

# 27.2 Pod Pending vì volume

Kiểm tra:

```bash
kubectl describe pod <pod>
```

Lỗi:

```text
FailedAttachVolume
```

Nguyên nhân:

```text
Volume đang attach node khác
RWO bị mount nhiều node
CSI lỗi
```

---

# 27.3 Multi-Attach Error

Ví dụ:

```text
Multi-Attach error for volume
```

Nguyên nhân:

PVC:

```text
ReadWriteOnce
```

Nhưng:

```text
Pod chạy trên 2 node
```

Giải pháp:

```text
Dùng RWX
Hoặc chạy StatefulSet
Hoặc dùng storage phù hợp
```

---

# 27.4 Data mất sau khi xóa PVC

Kiểm tra:

```bash
kubectl get pv
```

Nếu:

```yaml
reclaimPolicy: Delete
```

thì storage bị xóa.

Production nên cân nhắc:

```yaml
reclaimPolicy: Retain
```

---

# 28. So sánh nhanh các khái niệm

| Thành phần   | Vai trò                                 |
| ------------ | --------------------------------------- |
| Volume       | Mount storage vào Pod                   |
| emptyDir     | Storage tạm trong lifecycle Pod         |
| hostPath     | Mount filesystem Node                   |
| PV           | Storage resource trong cluster          |
| PVC          | Application request storage             |
| StorageClass | Template tạo PV                         |
| CSI          | Plugin kết nối storage backend          |
| StatefulSet  | Quản lý workload cần identity + storage |

---

# 29. Luồng storage hoàn chỉnh trong Kubernetes

Production thường có flow:

```text
Developer

   |
   |
Create PVC

   |
   |
StorageClass

   |
   |
CSI Driver

   |
   |
Cloud Storage

   |
   |
PersistentVolume

   |
   |
Pod mount

   |
   |
Application
```

---

# 30. Checklist cần nắm sau bài này

Sau khi học Volume, PVC và StorageClass cần trả lời được:

```text
Vì sao container filesystem không phù hợp lưu data?

Volume là gì?

emptyDir dùng khi nào?

hostPath có nhược điểm gì?

PV là gì?

PVC là gì?

Quan hệ PV và PVC?

StorageClass giải quyết vấn đề gì?

Static provisioning khác Dynamic provisioning thế nào?

CSI Driver làm nhiệm vụ gì?

reclaimPolicy Delete và Retain khác nhau?

RWO, RWX, ROX là gì?

Vì sao database thường dùng StatefulSet?

PVC Pending debug như thế nào?

Multi-Attach error do đâu?
```

---

# 31. Tóm tắt ngắn gọn

Kubernetes Pod là ephemeral:

```text
Pod chết
=
container mất
=
data có thể mất
```

Volume giúp:

```text
Container
    |
    v
Volume
    |
    v
Storage
```

Production dùng:

```text
PVC
 |
 |
StorageClass
 |
 |
CSI
 |
 |
Cloud Storage
```

Vai trò:

```text
PV
= storage thật

PVC
= request storage

StorageClass
= cách tạo storage

CSI
= cầu nối Kubernetes với storage provider
```

Mô hình production phổ biến:

```text
Application Pod

       |
       |
       v

PersistentVolumeClaim

       |
       |
       v

StorageClass

       |
       |
       v

Cloud Disk / NFS / Ceph / SAN
```

Hiểu chắc **Volume + PVC + StorageClass** là nền tảng để chạy các hệ thống production trên Kubernetes như:

```text
Database
Kafka
Elasticsearch
MinIO
Spark
Airflow
ML Training Pipeline
```

Sau bài này có thể học tiếp:

```text
05 - ConfigMap & Secret nâng cao
06 - Ingress và Gateway
07 - Resource Request/Limits
08 - HPA/VPA Autoscaling
09 - RBAC & Security
10 - Helm & Kubernetes Deployment Production
```

