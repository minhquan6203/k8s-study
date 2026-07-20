# Resource Requests, Limits và ResourceQuota trong Kubernetes: Quản lý CPU, Memory và tài nguyên Cluster

Ở các bài trước, chúng ta đã học:

* **Pod, Deployment, ReplicaSet**: quản lý vòng đời workload.
* **Service, DNS, Namespace**: giúp các application giao tiếp trong cluster.
* **Volume, PVC, StorageClass**: quản lý dữ liệu bền vững.

Tuy nhiên, còn một vấn đề rất quan trọng trong môi trường Kubernetes production:

```text
Cluster có bao nhiêu CPU?
Pod nào được phép dùng bao nhiêu RAM?
Nếu một application bị lỗi và ăn hết tài nguyên thì sao?
Làm sao giới hạn mỗi team chỉ được dùng một lượng tài nguyên nhất định?
```

Ví dụ:

Một Kubernetes cluster có:

```text
Worker Node 1

CPU:
16 cores

Memory:
64GB
```

Có hai application:

```text
Application A

Java backend


Application B

Machine Learning service
```

Nếu Application B bị lỗi:

```text
while(true){
    allocate memory
}
```

Memory tăng liên tục:

```text
2GB
10GB
30GB
60GB
```

Kết quả:

```text
Node hết RAM

        |
        v

Các Pod khác crash theo
```

Hiện tượng này gọi là:

```text
Noisy Neighbor Problem
```

Một workload ảnh hưởng đến workload khác.

Để giải quyết vấn đề này Kubernetes cung cấp:

```text
Resource Request

Resource Limit

ResourceQuota

LimitRange
```

---

# 1. Kubernetes quản lý tài nguyên như thế nào?

Mỗi node trong Kubernetes có tài nguyên:

```text
CPU
Memory
Ephemeral Storage
GPU
Network
```

Ví dụ:

```text
Worker Node

+----------------------+
| CPU: 16 cores        |
| RAM: 64Gi            |
| Disk: 500GB          |
+----------------------+
```

Pod chạy trên node sẽ tiêu thụ tài nguyên:

```text
Node

CPU 16 cores

        |
        |
        +---- Pod A
        |
        +---- Pod B
        |
        +---- Pod C
```

Kubernetes scheduler cần biết:

```text
Pod cần bao nhiêu tài nguyên?
Node nào còn đủ tài nguyên?
```

Đó là lý do có:

```text
requests
```

---

# 2. Resource Request là gì?

## 2.1 Khái niệm

**Resource request là lượng tài nguyên tối thiểu mà container yêu cầu Kubernetes đảm bảo cấp phát.**

Nói đơn giản:

```text
Request = Tôi cần ít nhất bao nhiêu tài nguyên để chạy
```

Ví dụ:

```yaml
resources:

  requests:

    cpu: "500m"

    memory: "512Mi"
```

Nghĩa là:

```text
Container cần tối thiểu:

CPU:
0.5 core

Memory:
512MB
```

---

# 2.2 Request ảnh hưởng đến Scheduler

Điểm rất quan trọng:

Kubernetes scheduler dựa vào request để quyết định đặt Pod ở node nào.

Ví dụ:

Cluster:

```
Node A

CPU:
4 cores

Memory:
8Gi


Node B

CPU:
8 cores

Memory:
32Gi
```

Pod yêu cầu:

```yaml
requests:

  cpu: "2"

  memory: "4Gi"
```

Scheduler kiểm tra:

Node A:

```
CPU còn:
4

RAM còn:
8Gi
```

Đủ.

Node B:

```
CPU còn:
8

RAM:
32Gi
```

Cũng đủ.

Scheduler chọn node phù hợp.

---

# 2.3 Request không phải giới hạn

Ví dụ:

```yaml
requests:

 cpu: "1"

 memory: "2Gi"
```

Không có nghĩa:

```text
Container chỉ được dùng 1 CPU và 2GB RAM
```

Nó chỉ có nghĩa:

```text
Kubernetes đảm bảo có ít nhất 1 CPU và 2GB RAM
```

Container có thể dùng nhiều hơn.

Ví dụ:

```
Request:

CPU:
1 core


Actual usage:

CPU:
3 cores
```

Nếu node còn tài nguyên:

```text
Container vẫn chạy bình thường
```

---

# 3. Resource Limit là gì?

## 3.1 Khái niệm

**Resource limit là giới hạn tối đa mà container được phép sử dụng.**

Ví dụ:

```yaml
resources:

  limits:

    cpu: "2"

    memory: "4Gi"
```

Nghĩa:

```text
Container tối đa:

CPU:
2 cores

Memory:
4GB
```

---

# 3.2 CPU Limit hoạt động như thế nào?

CPU có tính chất:

```text
compressible resource
```

Nghĩa là có thể throttle.

Ví dụ:

Limit:

```yaml
cpu: "2"
```

Application cần:

```
3 CPU
```

Kubernetes không kill container.

Nó giảm tốc:

```
Application request:

3 CPU

        |
        v

Allowed:

2 CPU
```

Hiện tượng:

```text
CPU throttling
```

---

# 3.3 Memory Limit hoạt động như thế nào?

Memory là:

```text
non-compressible resource
```

Không thể giảm tốc.

Ví dụ:

Limit:

```yaml
memory: "2Gi"
```

Application:

```
allocate 4GB
```

Kubernetes:

```
Memory vượt limit

        |
        v

OOMKilled
```

Pod:

```bash
kubectl get pods
```

Có thể thấy:

```
STATUS:

OOMKilled
```

---

# 4. CPU Unit trong Kubernetes

Kubernetes biểu diễn CPU bằng:

```text
core
millicore
```

---

## 4.1 CPU core

Ví dụ:

```yaml
cpu: "2"
```

nghĩa:

```text
2 CPU cores
```

---

## 4.2 Millicore

Ký hiệu:

```text
m
```

Ví dụ:

```yaml
cpu: "500m"
```

Nghĩa:

```text
500 millicores

= 0.5 CPU core
```

Bảng:

| Giá trị | Ý nghĩa  |
| ------- | -------- |
| 1000m   | 1 CPU    |
| 500m    | 0.5 CPU  |
| 250m    | 0.25 CPU |
| 100m    | 0.1 CPU  |

---

Ví dụ:

```yaml
resources:

 requests:

   cpu: "200m"

 limits:

   cpu: "1000m"
```

Nghĩa:

```
Minimum:
0.2 CPU

Maximum:
1 CPU
```

---

# 5. Memory Unit trong Kubernetes

Memory dùng:

```text
Byte
Ki
Mi
Gi
Ti
```

Thông dụng:

```text
Mi
Gi
```

---

Ví dụ:

```yaml
memory: "512Mi"
```

=

```text
512 Megabytes
```

---

```yaml
memory: "4Gi"
```

=

```text
4 Gigabytes
```

---

Bảng:

| Giá trị | Ý nghĩa |
| ------- | ------- |
| 256Mi   | 256 MB  |
| 512Mi   | 512 MB  |
| 1Gi     | 1 GB    |
| 8Gi     | 8 GB    |

---

# 6. Request và Limit trong Deployment

Ví dụ backend API:

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: backend


spec:

  replicas: 3


  template:

    spec:

      containers:

      - name: backend

        image: backend:v1


        resources:

          requests:

            cpu: "500m"

            memory: "512Mi"


          limits:

            cpu: "2"

            memory: "2Gi"
```

Ý nghĩa:

Mỗi Pod:

```
Minimum:

0.5 CPU
512MB RAM


Maximum:

2 CPU
2GB RAM
```

Deployment có:

```
replicas: 3
```

Tổng request:

```
CPU:

3 x 0.5

=
1.5 CPU


Memory:

3 x 512Mi

=
1536Mi
```

Scheduler phải tìm node có:

```
>= 1.5 CPU
>= 1.5GB RAM
```

---

# 7. QoS Class trong Kubernetes

Dựa vào request và limit, Kubernetes phân loại Pod thành:

```text
Guaranteed

Burstable

BestEffort
```

QoS ảnh hưởng khi node thiếu tài nguyên.

---

# 7.1 Guaranteed

Điều kiện:

Mọi container:

```text
request CPU == limit CPU

request memory == limit memory
```

Ví dụ:

```yaml
resources:

 requests:

   cpu: "2"

   memory: "4Gi"


 limits:

   cpu: "2"

   memory: "4Gi"
```

Pod:

```text
Guaranteed
```

Ưu tiên cao nhất.

---

# 7.2 Burstable

Có request nhưng limit lớn hơn.

Ví dụ:

```yaml
requests:

 cpu: "500m"

 memory: "1Gi"


limits:

 cpu: "2"

 memory: "4Gi"
```

Pod:

```
Burstable
```

Có thể burst khi cluster còn tài nguyên.

Đây là loại phổ biến nhất.

---

# 7.3 BestEffort

Không khai báo:

```yaml
resources:
```

Ví dụ:

```yaml
containers:

- name: app

  image: nginx
```

Pod:

```
BestEffort
```

Kubernetes cho dùng tài nguyên còn dư.

Nhưng khi node thiếu RAM:

```
BestEffort bị kill trước
```

---

# 8. Eviction khi Node thiếu tài nguyên

Ví dụ:

Node:

```
Memory:
64GB
```

Các Pod dùng:

```
70GB
```

Kubernetes phải giải phóng.

Quá trình:

```
Node pressure

        |
        v

Eviction

        |
        v

Kill Pod
```

Thứ tự ưu tiên:

```
BestEffort

    |

Burstable

    |

Guaranteed
```

---

# 9. ResourceQuota là gì?

## 9.1 Vấn đề

Namespace có thể chứa nhiều application.

Ví dụ:

Namespace:

```
team-a
```

Có:

```
backend
frontend
database
worker
```

Nếu không giới hạn:

Một team có thể deploy:

```
1000 Pods

100 CPU

500GB RAM
```

Làm ảnh hưởng toàn cluster.

---

ResourceQuota giúp:

```text
Giới hạn tổng tài nguyên của Namespace
```

---

# 10. ResourceQuota Manifest

Ví dụ:

Namespace:

```
dev
```

Quota:

```yaml
apiVersion: v1

kind: ResourceQuota


metadata:

  name: dev-quota


spec:

  hard:


    requests.cpu:

      "10"


    requests.memory:

      20Gi


    limits.cpu:

      "20"


    limits.memory:

      40Gi


    pods:

      "50"
```

Ý nghĩa:

Namespace này tối đa:

```
Pods:
50

CPU request:
10 cores

Memory request:
20GB

CPU limit:
20 cores

Memory limit:
40GB
```

---

# 11. ResourceQuota hoạt động như thế nào?

Ví dụ:

Quota:

```
CPU request:

10 cores
```

Hiện tại:

```
Pod A:

2 CPU


Pod B:

3 CPU
```

Đã dùng:

```
5 CPU
```

Deploy thêm:

```
6 CPU
```

Tổng:

```
11 CPU
```

Kubernetes reject:

```
exceeded quota
```

---

# 12. LimitRange là gì?

ResourceQuota giới hạn tổng namespace.

Nhưng:

```text
Một Pod mới tạo không khai báo resource thì sao?
```

Ví dụ:

Developer tạo:

```yaml
containers:

- image: nginx
```

Không có:

```yaml
resources:
```

Pod thành:

```
BestEffort
```

Không tốt.

---

LimitRange giúp đặt:

```text
Default request

Default limit

Minimum

Maximum
```

cho từng container.

---

# 13. LimitRange Manifest

Ví dụ:

```yaml
apiVersion: v1

kind: LimitRange


metadata:

  name: default-limit


spec:


  limits:


  - type: Container


    default:


      cpu: "1"

      memory: "1Gi"


    defaultRequest:


      cpu: "500m"

      memory: "512Mi"
```

Bây giờ:

Developer tạo:

```yaml
containers:

- name: nginx

  image: nginx
```

Kubernetes tự thêm:

```
Request:

500m CPU
512Mi RAM


Limit:

1 CPU
1Gi RAM
```

---

# 14. ResourceQuota + LimitRange kết hợp

Production thường dùng:

```text
Namespace

 |
 |
 +-- ResourceQuota
 |
 +-- LimitRange
```

Ví dụ:

```
Namespace team-a

Quota:

Max:
50 Pods
100 CPU
200GB RAM


LimitRange:

Default:

500m CPU
512Mi RAM
```

Kết quả:

```
Developer deploy app

        |
        v

LimitRange thêm resource

        |
        v

Quota kiểm tra

        |
        v

Scheduler đặt Pod
```

---

# 15. Kiểm tra Resource Usage

## 15.1 kubectl top

Cần metrics-server.

Node:

```bash
kubectl top nodes
```

Ví dụ:

```
NAME

worker-1

CPU:

3500m

MEMORY:

20Gi
```

---

Pod:

```bash
kubectl top pods
```

Ví dụ:

```
backend-abc

CPU:

300m

MEMORY:

700Mi
```

---

# 16. Troubleshooting Resource

---

# 16.1 Pod Pending

Kiểm tra:

```bash
kubectl describe pod <pod>
```

Lỗi:

```
Insufficient cpu
```

Nghĩa:

Node không đủ CPU request.

Ví dụ:

Pod cần:

```
4 CPU
```

Node còn:

```
2 CPU
```

Không schedule được.

---

# 16.2 OOMKilled

Xem:

```bash
kubectl describe pod <pod>
```

Có:

```
Reason:

OOMKilled
```

Nguyên nhân:

Memory vượt limit.

Ví dụ:

Limit:

```
1Gi
```

App:

```
2Gi
```

Fix:

Tăng:

```yaml
limits:

 memory: 2Gi
```

hoặc tối ưu app.

---

# 16.3 CPU Throttling

Triệu chứng:

```
Application chậm

CPU usage không cao
```

Nguyên nhân:

CPU limit quá thấp.

Ví dụ:

Limit:

```
500m
```

Application cần:

```
2 cores
```

Kubernetes throttle.

Fix:

Tăng CPU limit.

---

# 16.4 Quota exceeded

Lỗi:

```
exceeded quota
```

Kiểm tra:

```bash
kubectl describe quota
```

Ví dụ:

```
pods:

50/50
```

Cần:

* xóa workload không dùng
* tăng quota
* tạo namespace mới

---

# 17. Manifest đầy đủ Production

Ví dụ backend production:

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: api


spec:

  replicas: 3


  template:

    spec:

      containers:


      - name: api


        image: api:v1


        resources:


          requests:


            cpu: "500m"


            memory: "1Gi"



          limits:


            cpu: "2"


            memory: "4Gi"
```

Kèm:

```yaml
apiVersion: v1

kind: ResourceQuota


metadata:

  name: production-quota


spec:


  hard:


    pods:

      "100"


    requests.cpu:

      "50"


    requests.memory:

      100Gi
```

---

# 18. Bài Lab thực hành

## Lab 1: Tạo Deployment có Resource

Tạo:

```yaml
requests:

 cpu: 500m

 memory: 512Mi


limits:

 cpu: 1

 memory: 1Gi
```

Kiểm tra:

```bash
kubectl describe pod <pod>
```

---

# Lab 2: Quan sát Scheduler

Tạo Pod:

```yaml
requests:

 cpu: 20

 memory: 100Gi
```

Quan sát:

```bash
kubectl get pods
```

Kết quả:

```
Pending
```

Xem:

```bash
kubectl describe pod
```

---

# Lab 3: Test OOMKilled

Tạo Pod:

```yaml
limits:

 memory: 128Mi
```

Chạy app allocate RAM.

Quan sát:

```bash
kubectl get pods
```

Kết quả:

```
OOMKilled
```

---

# Lab 4: Tạo ResourceQuota

```bash
kubectl create namespace test
```

Apply quota.

Deploy nhiều Pod.

Quan sát:

```
exceeded quota
```

---

# 19. Checklist cần nắm sau bài này

Sau bài này cần hiểu:

```
Resource request là gì?

Resource limit là gì?

Scheduler dùng request hay limit?

CPU throttling là gì?

OOMKilled là gì?

CPU unit m và core?

Memory unit Mi/Gi?

QoS Class gồm những loại nào?

BestEffort bị kill khi nào?

ResourceQuota dùng để làm gì?

LimitRange khác ResourceQuota thế nào?

Pod Pending vì resource debug ra sao?

Production nên set request/limit như thế nào?
```

---

# 20. Tóm tắt ngắn gọn

Trong Kubernetes:

```
Request
=
tài nguyên tối thiểu Kubernetes đảm bảo


Limit
=
mức tối đa container được phép dùng
```

Ví dụ:

```yaml
resources:

 requests:

   cpu: 500m

   memory: 1Gi


 limits:

   cpu: 2

   memory: 4Gi
```

Nghĩa:

```
Scheduler đảm bảo:

>= 0.5 CPU
>= 1GB RAM


Container tối đa:

2 CPU
4GB RAM
```

---

Quản lý cấp cluster:

```
ResourceQuota
        |
        |
Giới hạn tổng namespace


LimitRange
        |
        |
Default resource cho từng container
```

Luồng hoàn chỉnh:

```
Developer tạo Pod

        |
        v

LimitRange áp default

        |
        v

ResourceQuota kiểm tra

        |
        v

Scheduler dùng request

        |
        v

Node chạy Pod

        |
        v

Limit kiểm soát runtime
```

Hiểu chắc **Resource Requests, Limits, Quota** là điều bắt buộc trước khi học:

```
06 - HPA/VPA Autoscaling
07 - Ingress
08 - RBAC & Security
09 - Helm
10 - Kubernetes Production Architecture
```

Vì toàn bộ việc **scale app, tối ưu chi phí cloud, tránh cluster chết vì thiếu tài nguyên** đều dựa trên phần này.
