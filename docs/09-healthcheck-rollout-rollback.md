# 09. Health Check, Rollout và Rollback trong Kubernetes: Cách Kubernetes đảm bảo ứng dụng luôn hoạt động ổn định

Ở các bài trước, ta đã học:

* **Pod, Deployment, ReplicaSet**: cách Kubernetes quản lý workload.
* **Service, DNS, Namespace**: cách các ứng dụng tìm thấy nhau trong cluster.
* **ConfigMap, Secret**: cách quản lý cấu hình và thông tin nhạy cảm.
* **Volume, PVC, StorageClass**: cách lưu trữ dữ liệu bền vững.
* **StatefulSet**: cách chạy các ứng dụng stateful như Kafka, Database.
* **RBAC, ServiceAccount**: cách kiểm soát quyền truy cập trong cluster.
* **Ingress, NodePort, LoadBalancer**: cách expose ứng dụng ra bên ngoài.

Tuy nhiên, một câu hỏi quan trọng vẫn còn:

> Kubernetes làm sao biết ứng dụng đang chạy có thực sự khỏe hay không?

Ví dụ:

Một Pod có trạng thái:

```text
STATUS: Running
```

nhưng bên trong ứng dụng có thể:

* Java Spring Boot bị deadlock.
* API trả HTTP 500 liên tục.
* Database connection bị mất.
* Application bị treo nhưng process vẫn còn chạy.
* Service nhận request nhưng không xử lý được.

Kubernetes chỉ nhìn thấy container process đang tồn tại.

Nó không biết ứng dụng bên trong có hoạt động đúng hay không.

Để giải quyết vấn đề này, Kubernetes cung cấp cơ chế:

```text
Health Check
    |
    +--> Liveness Probe
    |
    +--> Readiness Probe
    |
    +--> Startup Probe
```

Ngoài ra Kubernetes còn cung cấp:

```text
Rolling Update
    |
    +--> Deploy version mới an toàn

Rollback
    |
    +--> Quay lại version cũ khi có lỗi
```

Đây là nền tảng để xây dựng hệ thống:

* Zero downtime deployment
* Continuous Delivery
* Self-healing application
* Production reliability

---

# 1. Vấn đề: Container Running chưa chắc Application Healthy

Giả sử ta deploy một API backend:

```text
Deployment
    |
    +-- Pod backend-1
          |
          +-- Container API
```

Kubernetes kiểm tra:

```bash
docker ps
```

thấy container vẫn chạy:

```text
STATUS: Running
```

Kubernetes nghĩ:

```text
Container đang chạy => mọi thứ OK
```

Nhưng thực tế:

```text
GET /api/user

500 Internal Server Error
```

hoặc:

```text
Database connection timeout
```

hoặc:

```text
Application deadlock
```

Kubernetes không biết.

Vì vậy cần có cơ chế để Kubernetes hỏi:

> "Ứng dụng này có thật sự khỏe không?"

Đó chính là Probe.

---

# 2. Probe trong Kubernetes là gì?

Probe là một cơ chế Kubernetes dùng để kiểm tra trạng thái của container.

Kubernetes định kỳ gửi request hoặc thực hiện kiểm tra vào container.

Ví dụ:

```text
Kubernetes
     |
     | health check
     |
     v

Backend Container

/api/health
```

Nếu ứng dụng trả kết quả tốt:

```text
HTTP 200 OK
```

Kubernetes xem là healthy.

Nếu thất bại:

```text
HTTP 500
timeout
connection refused
```

Kubernetes sẽ xử lý tùy loại probe.

---

Kubernetes có 3 loại probe chính:

```text
1. Liveness Probe
   "Ứng dụng còn sống không?"

2. Readiness Probe
   "Ứng dụng đã sẵn sàng nhận traffic chưa?"

3. Startup Probe
   "Ứng dụng đã khởi động xong chưa?"
```

---

# 3. Liveness Probe

## 3.1 Liveness Probe là gì?

Liveness nghĩa là:

```text
Process còn sống không?
```

Mục tiêu của Liveness Probe:

> Phát hiện container bị treo hoặc không thể phục hồi.

Nếu Liveness Probe fail nhiều lần:

```text
Kubernetes kill container
        |
        v
Restart container
```

---

Ví dụ:

Một ứng dụng Java:

```text
Container vẫn chạy

PID 1 java process tồn tại

nhưng thread bị deadlock
```

Kubernetes thấy:

```text
container running
```

nhưng Liveness Probe:

```http
GET /health

timeout
```

=> Kubernetes restart container.

---

# 4. Các loại Liveness Probe

Kubernetes hỗ trợ nhiều cách kiểm tra.

Có 3 kiểu phổ biến:

```text
HTTP Probe
TCP Probe
Exec Probe
```

---

# 5. HTTP Liveness Probe

Đây là cách phổ biến nhất cho web application.

Ví dụ app có endpoint:

```text
GET /health
```

Response:

```json
{
  "status":"ok"
}
```

Deployment:

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
        - name: api
          image: backend:v1

          livenessProbe:
            httpGet:
              path: /health
              port: 8080

            initialDelaySeconds: 30
            periodSeconds: 10
```

Giải thích:

```yaml
httpGet:
```

Kubernetes gửi HTTP request:

```http
GET http://container:8080/health
```

---

## initialDelaySeconds

Ví dụ:

```yaml
initialDelaySeconds: 30
```

Nghĩa là:

```text
Sau khi container start
đợi 30 giây
rồi mới bắt đầu health check
```

Vì nhiều app cần thời gian startup.

Ví dụ:

Spring Boot:

```text
0s:
start JVM

10s:
load dependency

25s:
connect database

30s:
ready
```

Nếu check quá sớm:

```text
Kubernetes tưởng app chết
restart liên tục
```

---

## periodSeconds

Ví dụ:

```yaml
periodSeconds: 10
```

Nghĩa:

```text
Cứ mỗi 10 giây kiểm tra một lần
```

Flow:

```text
10s check
20s check
30s check
40s check
```

---

## failureThreshold

Ví dụ:

```yaml
failureThreshold: 3
```

Nghĩa:

```text
Fail liên tiếp 3 lần
mới restart container
```

Ví dụ:

```
10s fail
20s fail
30s fail

=> restart
```

Mục đích:

Tránh restart chỉ vì một lỗi mạng tạm thời.

---

# 6. TCP Liveness Probe

TCP Probe kiểm tra port có mở không.

Ví dụ:

```yaml
livenessProbe:
  tcpSocket:
    port: 8080

  initialDelaySeconds: 15
  periodSeconds: 10
```

Kubernetes thử:

```text
connect container:8080
```

Nếu:

```text
connection success
```

=> healthy

Nếu:

```text
connection refused
```

=> fail

---

TCP Probe phù hợp cho:

```text
Kafka
Redis
Database
TCP service
```

Ví dụ Kafka:

```text
port 9092
```

---

# 7. Exec Liveness Probe

Kubernetes chạy một command bên trong container.

Ví dụ:

```yaml
livenessProbe:

  exec:
    command:
      - cat
      - /tmp/healthy

  initialDelaySeconds: 5
  periodSeconds: 5
```

Kubernetes chạy:

```bash
cat /tmp/healthy
```

Nếu exit code:

```text
0
```

healthy.

Nếu:

```text
non-zero
```

fail.

---

Ví dụ:

```bash
health-check.sh
```

Script:

```bash
#!/bin/bash

check_database

if [ $? -eq 0 ]
then
    exit 0
else
    exit 1
fi
```

---

# 8. Readiness Probe

## 8.1 Readiness khác Liveness như thế nào?

Đây là phần rất quan trọng.

Liveness:

```text
Ứng dụng còn sống không?
```

Readiness:

```text
Ứng dụng đã sẵn sàng nhận request chưa?
```

---

Ví dụ:

Một Pod backend mới được tạo:

```text
Pod start
     |
     |
Loading model AI 5 phút
     |
     |
Connect database
     |
     |
Ready
```

Trong 5 phút đầu:

Container vẫn chạy.

Nhưng không nên nhận traffic.

Nếu không có Readiness:

```text
User request
      |
      v
Pod chưa ready
      |
      v
Error
```

---

Readiness Probe giúp:

```text
Pod chưa sẵn sàng
        |
        v
Không đưa vào Service endpoint
```

---

# 9. Ví dụ Readiness Probe

```yaml
readinessProbe:

  httpGet:
    path: /ready
    port: 8080

  initialDelaySeconds: 10
  periodSeconds: 5
```

Flow:

```
Pod start

        |
        v

Readiness check

        |
        +---- fail
        |
        v

Không nhận traffic


        |
        +---- success
        |
        v

Thêm vào Service endpoint
```

---

# 10. Liveness và Readiness kết hợp

Trong production thường dùng cả hai.

Ví dụ:

```yaml
containers:

- name: api
  image: backend:v1


  livenessProbe:

    httpGet:
      path: /health
      port: 8080


  readinessProbe:

    httpGet:
      path: /ready
      port: 8080
```

Ý nghĩa:

```
/health

App còn chạy không?


/ready

App có nhận request được chưa?
```

---

# 11. Startup Probe

## 11.1 Vấn đề với ứng dụng startup lâu

Một số app mất rất lâu để khởi động:

Ví dụ:

```text
Machine Learning Model

Load model 8GB

30 phút startup
```

Nếu dùng Liveness:

```
0s start

10s health check

fail

restart

loop vô hạn
```

Ứng dụng không bao giờ chạy được.

---

Startup Probe giải quyết vấn đề này.

---

# 12. Startup Probe hoạt động thế nào?

Khi có Startup Probe:

```text
Container start

        |
        v

Startup Probe chạy

        |
        |
   chưa pass

        |
        v

Không chạy Liveness


        |
        |
Startup pass

        |
        v

Bắt đầu Liveness + Readiness
```

---

Ví dụ:

```yaml
startupProbe:

  httpGet:
    path: /health
    port: 8080

  failureThreshold: 30
  periodSeconds: 10
```

Ý nghĩa:

```text
Cho app tối đa:

30 lần check

mỗi 10 giây

= 300 giây startup
```

---

# 13. So sánh 3 Probe

| Probe     | Câu hỏi                     | Fail thì            |
| --------- | --------------------------- | ------------------- |
| Liveness  | App còn sống không?         | Restart container   |
| Readiness | App nhận traffic được chưa? | Remove khỏi Service |
| Startup   | App đã start xong chưa?     | Restart container   |

Nhớ đơn giản:

```
Startup:
"Tôi đã khởi động xong chưa?"

Readiness:
"Tôi nhận request được chưa?"

Liveness:
"Tôi có bị chết không?"
```

---

# 14. Deployment Rolling Update

Bây giờ ta qua phần rollout.

Khi update application:

Ví dụ:

Version cũ:

```
backend:v1
```

Version mới:

```
backend:v2
```

Không nên:

```
Kill toàn bộ v1

Start v2
```

Vì sẽ downtime.

Kubernetes sử dụng:

```text
Rolling Update
```

---

# 15. Rolling Update là gì?

Rolling Update thay đổi version từng phần.

Ví dụ:

Ban đầu:

```
backend:v1

Pod 1
Pod 2
Pod 3
```

Update:

```
backend:v2
```

Kubernetes:

Bước 1:

```
Create Pod v2

Pod v1
Pod v1
Pod v1

Pod v2
```

Bước 2:

```
Remove Pod v1

Pod v1
Pod v1

Pod v2
Pod v2
```

Bước 3:

```
Remove hết v1

Pod v2
Pod v2
Pod v2
```

Không downtime.

---

# 16. Deployment Strategy

Deployment có:

```yaml
strategy:
  type: RollingUpdate
```

Mặc định là:

```text
RollingUpdate
```

---

Có 2 strategy:

```
RollingUpdate

Recreate
```

---

# 17. Recreate Strategy

Ví dụ:

```yaml
strategy:
  type: Recreate
```

Flow:

```
Stop tất cả Pod cũ

        |
        v

Start Pod mới
```

Có downtime.

Dùng khi:

* App không chạy được nhiều version cùng lúc.
* Database migration đặc biệt.
* Legacy application.

---

# 18. maxSurge và maxUnavailable

Trong Rolling Update:

```yaml
strategy:
  rollingUpdate:

    maxSurge: 1

    maxUnavailable: 1
```

---

## maxSurge

Số Pod được phép tạo thêm vượt replicas.

Ví dụ:

```yaml
replicas: 3

maxSurge: 1
```

Trong update:

```
Có thể có:

3 pod cũ

+

1 pod mới

=

4 pod
```

---

## maxUnavailable

Số Pod được phép unavailable.

Ví dụ:

```yaml
maxUnavailable: 1
```

Nghĩa:

Trong quá trình update:

```
Ít nhất phải có 2 Pod chạy
```

---

# 19. Rollout trong Kubernetes

Rollout là quá trình thay đổi version Deployment.

Ví dụ:

```bash
kubectl set image deployment/backend \
api=backend:v2
```

Kubernetes tạo ReplicaSet mới.

Kiểm tra:

```bash
kubectl rollout status deployment/backend
```

---

Xem lịch sử:

```bash
kubectl rollout history deployment/backend
```

Ví dụ:

```
REVISION
1 backend:v1
2 backend:v2
```

---

# 20. Rollback

Giả sử deploy:

```
backend:v2
```

nhưng lỗi:

```
HTTP 500
CrashLoopBackOff
```

Ta rollback:

```bash
kubectl rollout undo deployment/backend
```

Kubernetes quay lại:

```
backend:v1
```

---

Rollback về revision cụ thể:

```bash
kubectl rollout undo deployment/backend \
--to-revision=1
```

---

# 21. Kiểm tra rollout lỗi

Ví dụ:

```bash
kubectl set image deployment/backend \
api=backend:notfound
```

Kubernetes tạo Pod:

```
backend:notfound
```

Nhưng:

```
ImagePullBackOff
```

Rollout bị treo.

Kiểm tra:

```bash
kubectl rollout status deployment/backend
```

Có thể thấy:

```
Waiting for deployment to finish
```

---

Debug:

```bash
kubectl get pods

kubectl describe pod <pod>
```

---

# 22. Deployment Revision History

Mỗi lần update:

Deployment tạo ReplicaSet mới.

Ví dụ:

```
Deployment backend

ReplicaSet v1
    |
    +-- Pod v1


ReplicaSet v2
    |
    +-- Pod v2
```

Xem:

```bash
kubectl get rs
```

Ví dụ:

```
NAME              DESIRED
backend-v1        0
backend-v2        3
```

---

# 23. Manifest Production đầy đủ

Ví dụ backend production:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: backend

spec:

  replicas: 3


  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxSurge: 1

      maxUnavailable: 1


  template:

    spec:

      containers:

      - name: api

        image: backend:v2


        ports:

        - containerPort: 8080



        startupProbe:

          httpGet:

            path: /health

            port: 8080

          failureThreshold: 30

          periodSeconds: 10



        readinessProbe:

          httpGet:

            path: /ready

            port: 8080

          periodSeconds: 5



        livenessProbe:

          httpGet:

            path: /health

            port: 8080

          periodSeconds: 10
```

---

# 24. Chuỗi lệnh thực hành

## Deploy app

```bash
kubectl apply -f backend.yaml
```

---

## Kiểm tra Pod

```bash
kubectl get pods
```

---

## Xem probe status

```bash
kubectl describe pod <pod-name>
```

---

## Update version

```bash
kubectl set image deployment/backend \
api=backend:v2
```

---

## Theo dõi rollout

```bash
kubectl rollout status deployment/backend
```

---

## Xem history

```bash
kubectl rollout history deployment/backend
```

---

## Rollback

```bash
kubectl rollout undo deployment/backend
```

---

# 25. Troubleshooting Health Check

## Pod restart liên tục

Kiểm tra:

```bash
kubectl describe pod <pod>
```

Xem:

```
Liveness probe failed
```

Nguyên nhân:

```
initialDelaySeconds quá thấp

endpoint health sai

app startup lâu

port sai
```

---

## Service không nhận traffic

Kiểm tra:

```bash
kubectl get endpoints
```

Nếu:

```
<none>
```

Có thể:

```
Readiness probe fail
```

Pod chưa được đưa vào Service.

---

## Deployment không rollout được

Kiểm tra:

```bash
kubectl rollout status deploy/backend
```

Sau đó:

```bash
kubectl get pods
```

Các lỗi thường gặp:

```
ImagePullBackOff

CrashLoopBackOff

Readiness probe failed

Resource thiếu
```

---

# 26. Bài lab tự làm

## Lab 1: Tạo app có health check

Tạo Deployment:

```
backend:v1
```

Có:

```
livenessProbe
readinessProbe
```

---

## Lab 2: Làm readiness fail

Sửa endpoint:

```
/ready
```

thành:

```
/wrong
```

Quan sát:

```bash
kubectl get endpoints
```

Kết quả:

```
Không còn Pod endpoint
```

---

## Lab 3: Rolling Update

Deploy:

```
v1
```

Update:

```
v2
```

Quan sát:

```bash
kubectl get pods -w
```

Thấy:

```
Pod cũ giảm dần
Pod mới tăng dần
```

---

## Lab 4: Rollback

Deploy image lỗi:

```
backend:error
```

Sau đó:

```bash
kubectl rollout undo deployment/backend
```

Kiểm tra:

```bash
kubectl get pods
```

---

# 27. Checklist cần nắm sau bài này

Sau bài này cần hiểu:

```
Probe là gì?

Liveness khác Readiness thế nào?

Startup Probe dùng khi nào?

Health check ảnh hưởng Service như thế nào?

Rolling Update hoạt động ra sao?

maxSurge là gì?

maxUnavailable là gì?

Deployment tạo ReplicaSet mới khi nào?

Rollout history lưu ở đâu?

Rollback hoạt động thế nào?

Debug rollout fail bằng lệnh nào?
```

---

# 28. Tóm tắt ngắn gọn

Kubernetes không chỉ chạy container.

Nó còn phải đảm bảo:

```text
Container khỏe
Application sẵn sàng
Update không downtime
Có thể quay lại version cũ
```

Ba Probe quan trọng:

```
Startup Probe:
Ứng dụng đã khởi động xong chưa?


Readiness Probe:
Có nhận traffic được chưa?


Liveness Probe:
Có cần restart không?
```

Deployment lifecycle:

```
Deploy v1

      |
      v

Update v2

      |
      v

Rolling Update

      |
      v

Health Check

      |
      +---- OK
      |
      +---- Fail

              |
              v

          Rollback v1
```

Hiểu chắc Health Check, Rollout và Rollback là nền tảng để triển khai Kubernetes production, đặc biệt trong môi trường CI/CD, GitOps và microservices.
