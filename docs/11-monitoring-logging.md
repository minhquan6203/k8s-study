# 11. Monitoring và Logging trong Kubernetes: Quan sát, đo lường và xử lý sự cố trong Cluster

Ở các bài trước, chúng ta đã học cách Kubernetes:

* Deploy application bằng Deployment.
* Expose application bằng Service.
* Quản lý configuration bằng ConfigMap và Secret.
* Lưu trữ dữ liệu bằng Volume và PVC.
* Chạy workload stateful bằng StatefulSet.
* Phân quyền bằng RBAC.
* Quản lý package bằng Helm.

Tuy nhiên, khi hệ thống chạy thực tế, một câu hỏi quan trọng xuất hiện:

> Làm sao biết application đang khỏe hay đang lỗi?

Ví dụ:

```text
Application không trả response.

Nguyên nhân có thể là:

- Pod bị crash
- CPU quá cao
- Memory leak
- Database chậm
- Network lỗi
- Disk đầy
- Service không route được
- Node bị quá tải
```

Nếu chỉ dùng:

```bash
kubectl get pods
```

ta chỉ biết:

```text
Running
```

Nhưng:

```text
Running != Healthy
```

Một Pod có thể:

```text
Running
Ready 1/1

nhưng:

- Application bên trong bị deadlock
- API trả lỗi 500
- Query database timeout
- Memory tăng dần
```

Vì vậy production Kubernetes bắt buộc cần:

```text
Monitoring
+
Logging
+
Alerting
```

---

# 1. Monitoring là gì?

## 1.1 Khái niệm Monitoring

Monitoring là quá trình thu thập và phân tích các chỉ số (metrics) của hệ thống theo thời gian.

Ví dụ:

```text
CPU usage

Memory usage

Network traffic

Disk usage

Request per second

Response latency

Error rate

Database connection
```

Monitoring trả lời các câu hỏi:

```text
Hệ thống đang khỏe không?

Traffic hiện tại bao nhiêu?

Có dấu hiệu quá tải không?

Khi nào cần scale?

Lỗi bắt đầu từ đâu?
```

---

# 1.2 Monitoring khác Logging như thế nào?

Hai khái niệm này thường bị nhầm.

## Monitoring

Tập trung vào:

```text
Số liệu tổng quan
```

Ví dụ:

```text
CPU = 85%

Memory = 90%

Request = 5000 req/s

Error rate = 5%
```

Nó trả lời:

> "Có vấn đề xảy ra không?"

---

## Logging

Tập trung vào:

```text
Chi tiết sự kiện
```

Ví dụ log application:

```text
2026-07-27 10:00:01
ERROR Database connection timeout

2026-07-27 10:00:05
ERROR Failed login user_id=123
```

Nó trả lời:

> "Vấn đề xảy ra vì sao?"

---

Ví dụ:

Monitoring phát hiện:

```text
API error rate tăng lên 20%
```

Sau đó dùng Logging để tìm:

```text
Database connection timeout
```

Kết hợp:

```text
Monitoring
     |
     v
Phát hiện vấn đề

Logging
     |
     v
Tìm nguyên nhân
```

---

# 2. Vì sao Kubernetes cần Monitoring?

Trong hệ thống truyền thống:

```text
Server
 |
Application
```

Có thể SSH vào server:

```bash
top

tail -f app.log
```

Nhưng Kubernetes khác:

```text
Cluster

 |
 +-- Node 1
 |
 +-- Node 2
 |
 +-- Node 3


Pods liên tục thay đổi
```

Pod có thể:

```text
Create

Destroy

Move

Restart

Scale
```

Không thể dựa vào:

```text
SSH vào một server cố định
```

---

Kubernetes cần monitoring vì:

## 1. Dynamic environment

Pod không cố định:

```text
backend-7d8f9d-x123

backend-7d8f9d-x456
```

Ngày mai:

```text
backend-8abc9f-x789
```

---

## 2. Scale tự động

Ví dụ:

Ban ngày:

```text
3 Pods
```

Ban đêm:

```text
1 Pod
```

Monitoring giúp Kubernetes biết:

```text
CPU tăng

↓

Scale thêm Pod
```

---

## 3. Debug nhanh

Khi user báo:

```text
Website chậm
```

Cần biết:

```text
CPU?
Memory?
Network?
Database?
Pod restart?
```

---

# 3. Các loại dữ liệu cần monitor trong Kubernetes

Một Kubernetes production thường monitor 4 tầng:

```text
Application Layer

        |

Container Layer

        |

Node Layer

        |

Cluster Layer
```

---

# 4. Application Metrics

Đây là metric từ chính application.

Ví dụ API backend:

```text
Request count

Latency

HTTP status code

Error rate

Active user

Database query time
```

Ví dụ:

Prometheus metric:

```text
http_requests_total 500000

http_request_duration_seconds 0.2
```

---

Application thường expose metric endpoint:

Ví dụ:

```text
http://app:8080/metrics
```

Prometheus sẽ scrape endpoint này.

---

Ví dụ Python FastAPI:

```python
from prometheus_client import Counter

REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests"
)
```

Expose:

```text
/metrics
```

---

# 5. Container Metrics

Container cần monitor:

```text
CPU

Memory

Network

Filesystem
```

Ví dụ:

Pod:

```text
backend

CPU:
800m / 1000m


Memory:
900Mi / 1Gi
```

Có thể phát hiện:

```text
CPU throttling

Memory leak

OOMKilled
```

---

# 6. Node Metrics

Node là máy vật lý hoặc VM chạy Kubernetes.

Monitor:

```text
CPU

RAM

Disk

Network

Filesystem

Kernel
```

Ví dụ:

Node:

```text
worker-node-01

CPU:
95%

Memory:
85%

Disk:
90%
```

Có thể dẫn đến:

```text
Pod Pending

Pod Evicted

Scheduling fail
```

---

# 7. Cluster Metrics

Monitor Kubernetes component:

Ví dụ:

## API Server

```text
Request latency

Request error
```

---

## Scheduler

```text
Scheduling failure
```

---

## Controller Manager

```text
Controller errors
```

---

## etcd

Rất quan trọng:

```text
Database của Kubernetes
```

Monitor:

```text
Disk latency

Leader election

Request latency
```

---

# 8. Kiến trúc Monitoring Kubernetes phổ biến

Stack phổ biến nhất:

```text
Prometheus

+

Grafana

+

Alertmanager

+

Node Exporter

+

kube-state-metrics
```

Kiến trúc:

```
                Kubernetes Cluster


        +-----------------------+
        |                       |
        |   Application Pods    |
        |                       |
        +-----------+-----------+
                    |
                    |
              /metrics endpoint

                    |
                    v

             Prometheus Server

                    |
        +-----------+-----------+

        |                       |

    Grafana              Alertmanager

        |                       |

 Dashboard              Notification

                         |
                 Slack / Email / PagerDuty
```

---

# 9. Prometheus là gì?

Prometheus là hệ thống monitoring mã nguồn mở được thiết kế cho cloud native.

Nó là thành phần trung tâm trong Kubernetes monitoring.

---

Prometheus làm 3 việc chính:

## 1. Pull metrics

Prometheus không chờ application gửi dữ liệu.

Nó chủ động lấy:

```text
Scrape
```

Ví dụ:

Mỗi 15 giây:

```
Prometheus

      |

      v

http://backend:8080/metrics
```

---

## 2. Lưu time-series database

Prometheus lưu dữ liệu dạng:

```text
Metric

+

Timestamp

+

Labels
```

Ví dụ:

```
http_requests_total

method="GET"

status="200"

time="10:00"
```

---

## 3. Query bằng PromQL

Prometheus có ngôn ngữ:

```text
PromQL
```

Ví dụ:

CPU trung bình:

```promql
rate(
container_cpu_usage_seconds_total[5m]
)
```

---

# 10. Grafana là gì?

Prometheus lưu dữ liệu.

Nhưng con người khó đọc:

```text
http_requests_total 23847291
```

Grafana giúp visualize.

Ví dụ dashboard:

```
+----------------------------+

CPU Usage

██████████ 85%


Memory

███████ 70%


Requests/sec

5000


Error Rate

2%

+----------------------------+
```

---

Grafana có:

```text
Dashboard

Panel

Graph

Table

Alert
```

---

# 11. Alertmanager là gì?

Monitoring mà không alert thì chưa đủ.

Ví dụ:

Prometheus thấy:

```text
CPU > 90%
trong 5 phút
```

Nó gửi alert:

```
Prometheus

      |

      v

Alertmanager

      |

      +---- Slack

      +---- Email

      +---- PagerDuty
```

---

Ví dụ alert:

```yaml
alert:
  name: HighCPU

expr:
  cpu_usage > 90

for:
  5m
```

Ý nghĩa:

```text
CPU cao hơn 90%

liên tục 5 phút

=> gửi cảnh báo
```

---

# 12. Node Exporter

Node Exporter thu thập metric của máy host.

Ví dụ:

```text
CPU

Memory

Disk

Network
```

Kiến trúc:

```
Node

 |
 |
Node Exporter

 |
 |
Prometheus
```

---

Node Exporter thường chạy dạng:

```text
DaemonSet
```

vì:

```text
Mỗi node cần một exporter
```

Ví dụ:

```
worker-1
 |
 node-exporter


worker-2
 |
 node-exporter


worker-3
 |
 node-exporter
```

---

# 13. kube-state-metrics

Một thành phần rất quan trọng.

Nó lấy thông tin từ Kubernetes API.

Ví dụ:

```text
Pod status

Deployment replicas

Node status

PVC status

Job status
```

Ví dụ:

Deployment:

```yaml
replicas: 5
```

Nhưng thực tế:

```text
ready replicas: 3
```

kube-state-metrics expose:

```
kube_deployment_status_replicas_available 3
```

---

# 14. Metrics Server

Metrics Server là component Kubernetes dùng cho:

```text
kubectl top

Horizontal Pod Autoscaler
```

Ví dụ:

```bash
kubectl top pods
```

Output:

```
NAME          CPU     MEMORY

backend-1     500m    300Mi

backend-2     700m    400Mi
```

---

Metrics Server khác Prometheus:

| Metrics Server      | Prometheus          |
| ------------------- | ------------------- |
| Kubernetes built-in | Monitoring platform |
| CPU/Memory cơ bản   | Metric đa dạng      |
| Cho HPA             | Dashboard           |
| Không lưu lâu       | Lưu time-series     |

---

# 15. Cài Monitoring Stack bằng Helm

Cách phổ biến nhất:

Dùng:

```text
kube-prometheus-stack
```

Chart này bao gồm:

```text
Prometheus

Grafana

Alertmanager

Node Exporter

kube-state-metrics
```

---

Thêm repo:

```bash
helm repo add prometheus-community \
https://prometheus-community.github.io/helm-charts
```

Update:

```bash
helm repo update
```

Install:

```bash
helm install monitoring \
prometheus-community/kube-prometheus-stack \
-n monitoring \
--create-namespace
```

---

Kiểm tra:

```bash
kubectl get pods -n monitoring
```

Kết quả:

```
prometheus
grafana
alertmanager
node-exporter
kube-state-metrics
```

---

# 16. Kubernetes Logging Architecture: Thu thập và quản lý log trong Cluster

Tuy nhiên, metrics chỉ cho chúng ta biết:

```text
"Có vấn đề xảy ra"
```

Nhưng chưa trả lời được:

```text
"Tại sao vấn đề xảy ra?"
```

Ví dụ:

Monitoring phát hiện:

```text
API error rate tăng từ 1% lên 20%
```

Nhưng nguyên nhân có thể là:

```text
Database timeout

Authentication service lỗi

Bug trong code

Memory leak

Network failure
```

Lúc này cần đến **Logging**.

---

# 16.1 Logging là gì?

Logging là quá trình thu thập, lưu trữ và phân tích các message mà application, container hoặc Kubernetes component sinh ra.

Ví dụ application:

```text
2026-07-27 10:20:01 INFO User login success

2026-07-27 10:20:05 ERROR Database connection timeout

2026-07-27 10:20:10 WARN Retry request
```

Log giúp trả lời:

```text
Request nào lỗi?

User nào bị ảnh hưởng?

Service nào gây lỗi?

Lỗi xảy ra lúc nào?

Stack trace là gì?
```

---

# 16.2 Logging trong Kubernetes khác server truyền thống thế nào?

Trong hệ thống truyền thống:

```text
Server

 |
 |
Application

 |
 |
/var/log/app.log
```

Admin có thể SSH:

```bash
tail -f app.log
```

---

Nhưng Kubernetes:

```text
Cluster

 |
 +---- Node 1
 |
 |      +-- Pod A
 |      +-- Pod B
 |
 +---- Node 2
 |
        +-- Pod C
```

Pod có thể:

```text
Restart

Move node

Scale

Delete
```

Không thể:

```bash
ssh server
tail log
```

vì ngày mai Pod có thể nằm node khác.

---

Vì vậy Kubernetes logging thường dùng mô hình:

```text
Application

     |

Container stdout/stderr

     |

Node log collector

     |

Central Logging System

     |

Search + Dashboard
```

---

# 17. Kubernetes Logging Architecture

Kiến trúc phổ biến:

```text
                 Kubernetes Cluster


+------------------------------------------------+

 Node 1

  Pod A
    |
    | stdout
    |
    v

 Container Runtime Log


    |
    v

 Fluent Bit


+------------------------------------------------+


 Node 2

  Pod B
    |
    v

 Fluent Bit


+------------------------------------------------+

                |
                |
                v

          Log Storage

     +-------------------+

     | Elasticsearch     |
     | Loki              |
     | OpenSearch        |

     +-------------------+

                |
                v

             Dashboard

        Kibana / Grafana
```

---

# 18. Container Log hoạt động như thế nào?

Trong Kubernetes, ứng dụng thường không ghi log vào file.

Best practice:

```text
Write log to stdout/stderr
```

Ví dụ Python:

```python
print("User login success")
```

hoặc:

```python
logger.error(
    "Database timeout"
)
```

Output:

```text
stdout
```

Container runtime sẽ lưu lại.

---

Kiểm tra log:

```bash
kubectl logs <pod-name>
```

Ví dụ:

```bash
kubectl logs backend-7d8f9d-abc
```

Output:

```text
Application started

Listening port 8080

Database connected
```

---

# 18.1 Log của Pod nhiều container

Một Pod có thể có nhiều container:

```text
Pod

 |
 +-- application container

 |
 +-- sidecar container
```

Ví dụ:

```yaml
containers:

- name: app

- name: log-agent
```

Khi đó cần chỉ rõ:

```bash
kubectl logs pod-name \
-c app
```

---

# 18.2 Xem log realtime

Giống:

```bash
tail -f
```

Trong Kubernetes:

```bash
kubectl logs -f pod-name
```

Ví dụ:

```bash
kubectl logs -f backend-123
```

---

# 18.3 Xem log container trước khi restart

Một lỗi phổ biến:

Pod:

```text
CrashLoopBackOff
```

Container hiện tại đã restart.

Log hiện tại:

```bash
kubectl logs pod-name
```

có thể không có nhiều thông tin.

Dùng:

```bash
kubectl logs pod-name --previous
```

Để xem log của container trước lần restart.

---

# 19. Log Rotation trong Kubernetes

Một vấn đề:

Nếu application ghi log liên tục:

```text
1GB/day

10GB/day

100GB/day
```

Disk node sẽ đầy.

Kubernetes không giữ log vô hạn.

Container runtime thường có cơ chế:

```text
Log rotation
```

Ví dụ:

```text
container.log

container.log.1

container.log.2
```

---

Production cần:

```text
Log retention policy

Log compression

Log aggregation
```

---

# 20. Centralized Logging

Trong production:

Không nên:

```text
kubectl logs từng Pod
```

vì:

```text
100 Pods

1000 Pods

10000 Pods
```

không thể quản lý.

Cần hệ thống:

```text
Central Logging
```

---

Mô hình:

```text
Application Pods

      |
      v

Log Collector

      |
      v

Central Storage

      |
      v

Search Dashboard
```

---

Các stack phổ biến:

```text
ELK Stack

EFK Stack

Loki + Grafana
```

---

# 21. ELK Stack

ELK là một logging stack rất phổ biến.

Bao gồm:

```text
Elasticsearch

Logstash

Kibana
```

---

# 21.1 Elasticsearch

Elasticsearch là nơi lưu trữ và tìm kiếm log.

Đặc điểm:

```text
Distributed search engine

Full-text search

Near real-time query
```

Ví dụ lưu:

```json
{
 "service":"backend",
 "level":"ERROR",
 "message":"Database timeout",
 "timestamp":"2026-07-27"
}
```

Có thể query:

```text
service=backend AND level=ERROR
```

---

# 21.2 Logstash

Logstash là log processor.

Nhiệm vụ:

```text
Receive log

Parse log

Transform

Send to storage
```

Ví dụ:

Raw log:

```text
ERROR Database timeout user=123
```

Transform:

```json
{
 "level":"ERROR",
 "error":"Database timeout",
 "user":"123"
}
```

---

# 21.3 Kibana

Kibana là dashboard cho Elasticsearch.

Có thể:

```text
Search log

Create dashboard

Create visualization
```

Ví dụ dashboard:

```text
Error count

Top errors

Request latency

Failed login
```

---

# 22. EFK Stack

EFK thay Logstash bằng Fluentd/Fluent Bit.

Bao gồm:

```text
Elasticsearch

Fluentd

Kibana
```

---

Kiến trúc:

```text
Pod

 |

stdout

 |

Fluent Bit / Fluentd

 |

Elasticsearch

 |

Kibana
```

---

Ưu điểm:

```text
Nhẹ hơn Logstash

Cloud native

Phù hợp Kubernetes
```

---

# 23. Fluent Bit là gì?

Fluent Bit là log collector rất phổ biến trong Kubernetes.

Nó thường chạy dạng:

```text
DaemonSet
```

Lý do:

Mỗi node cần một agent.

Ví dụ:

```text
Node 1

 |
 Fluent Bit


Node 2

 |
 Fluent Bit


Node 3

 |
 Fluent Bit
```

---

Fluent Bit:

Đọc:

```text
/var/log/containers/*.log
```

Sau đó gửi:

```text
Elasticsearch

Loki

Kafka

Cloud Logging
```

---

# 24. Loki + Grafana Logging Stack

Một lựa chọn hiện đại hơn ELK:

```text
Loki

+

Promtail

+

Grafana
```

---

Kiến trúc:

```text
Pod logs

   |

Promtail

   |

Loki

   |

Grafana
```

---

# 24.1 Loki

Loki là log storage của Grafana Labs.

Khác Elasticsearch:

Elasticsearch:

```text
Index toàn bộ nội dung log
```

Loki:

```text
Index metadata

Store log content
```

Ví dụ:

Loki index:

```text
app=backend
namespace=prod
level=error
```

Log:

```text
Database timeout
```

---

Ưu điểm Loki:

```text
Nhẹ hơn Elasticsearch

Ít tốn storage

Tích hợp Grafana tốt
```

---

# 24.2 Promtail

Promtail là log collector.

Nhiệm vụ:

```text
Read log

Add labels

Send Loki
```

Ví dụ:

Input:

```text
ERROR database timeout
```

Add label:

```text
namespace=production

app=backend
```

Gửi:

```text
Loki
```

---

# 25. Kubernetes Logging Best Practices

## 25.1 Log dạng structured JSON

Không nên:

```
User login failed
```

Nên:

```json
{
 "level":"ERROR",
 "service":"auth",
 "user_id":123,
 "message":"login failed"
}
```

Lợi ích:

```text
Search dễ hơn

Parse dễ hơn

Alert dễ hơn
```

---

# 25.2 Không log secret

Sai:

```text
password=123456
token=abcdef
```

Vì log có thể được:

```text
team khác đọc

backup

export
```

---

Không log:

```text
Password

API key

JWT token

Private key
```

---

# 25.3 Gắn request ID

Microservice:

```text
Frontend

 |

Backend

 |

Payment

 |

Database
```

Một request đi qua nhiều service.

Cần:

```text
request_id
```

Ví dụ:

Frontend:

```json
{
 "request_id":"abc123"
}
```

Backend:

```json
{
 "request_id":"abc123",
 "message":"payment failed"
}
```

Dễ trace.

---

# 26. Monitoring + Logging kết hợp

Production không dùng riêng lẻ.

Ví dụ:

## Bước 1

Prometheus phát hiện:

```text
HTTP Error rate > 10%
```

---

## Bước 2

Alertmanager gửi:

```text
API Error High
```

---

## Bước 3

Engineer mở Grafana:

Thấy:

```text
backend pod error tăng
```

---

## Bước 4

Query Loki:

```text
{app="backend"} |= "ERROR"
```

Kết quả:

```text
Database connection timeout
```

---

Luồng hoàn chỉnh:

```text
Monitoring

    |
    v

Detect problem

    |
    v

Alert

    |
    v

Logging

    |
    v

Find root cause
```

---

# 27. Cài Loki Stack bằng Helm

Thêm repo:

```bash
helm repo add grafana \
https://grafana.github.io/helm-charts
```

Update:

```bash
helm repo update
```

Install Loki:

```bash
helm install loki \
grafana/loki-stack \
-n monitoring
```

Kiểm tra:

```bash
kubectl get pods -n monitoring
```

---

Có thể thấy:

```text
loki

promtail

grafana
```

---

# 28. Cài Fluent Bit bằng Helm

Thêm repo:

```bash
helm repo add fluent \
https://fluent.github.io/helm-charts
```

Install:

```bash
helm install fluent-bit \
fluent/fluent-bit \
-n logging \
--create-namespace
```

Kiểm tra:

```bash
kubectl get pods -n logging
```

---

# 29. Debug Logging Kubernetes

## 29.1 Pod không có log

Kiểm tra:

```bash
kubectl logs pod-name
```

Nếu:

```text
No logs available
```

Kiểm tra:

```bash
kubectl describe pod pod-name
```

---

Nguyên nhân:

```text
Application không ghi stdout

Container chưa chạy

Sai container name
```

---

# 29.2 Pod CrashLoopBackOff

Xem:

```bash
kubectl logs pod-name --previous
```

Thường thấy:

```text
Application exception

Configuration error

Database connection failed
```

---

# 29.3 Fluent Bit không lấy được log

Kiểm tra:

```bash
kubectl logs fluent-bit-pod -n logging
```

Kiểm tra:

```bash
kubectl describe daemonset fluent-bit -n logging
```

---

# 29.4 Elasticsearch đầy disk

Triệu chứng:

```text
Cannot index document
Disk watermark exceeded
```

Giải pháp:

```text
Increase storage

Delete old logs

Configure retention
```

---

# 30. Lab Monitoring + Logging

## Lab 1: Cài Prometheus Stack

```bash
helm install monitoring \
prometheus-community/kube-prometheus-stack \
-n monitoring \
--create-namespace
```

Kiểm tra:

```bash
kubectl get pods -n monitoring
```

---

## Lab 2: Kiểm tra Metrics

```bash
kubectl top nodes

kubectl top pods
```

---

## Lab 3: Cài Loki

```bash
helm install loki \
grafana/loki-stack \
-n monitoring
```

---

## Lab 4: Deploy app tạo log

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: logger-app

spec:
  replicas: 1

  template:
    spec:
      containers:

      - name: app
        image: busybox

        command:
        - sh
        - -c

        - |
          while true;
          do
            echo "hello kubernetes"
            sleep 5
          done
```

---

Xem log:

```bash
kubectl logs logger-app-xxx
```

---

# 31. Checklist cần nắm sau bài Monitoring & Logging

Sau bài này cần hiểu:

```text
Monitoring là gì?

Logging là gì?

Metric khác Log thế nào?

Prometheus làm gì?

Grafana làm gì?

Alertmanager làm gì?

Node Exporter dùng để làm gì?

kube-state-metrics là gì?

Metrics Server khác Prometheus thế nào?

Container log nằm ở đâu?

kubectl logs hoạt động thế nào?

Fluent Bit là gì?

ELK Stack là gì?

EFK Stack là gì?

Loki Stack là gì?

Centralized Logging là gì?

Vì sao production cần monitoring + logging?
```

---

# 32. Tổng kết

Một Kubernetes production cần ba lớp quan sát:

```text
                Observability


        +-------------------+

        |    Monitoring     |

        | CPU               |

        | Memory            |

        | Request           |

        +-------------------+

                 |

                 |

        +-------------------+

        |     Logging       |

        | Error             |

        | Exception         |

        | Trace             |

        +-------------------+

                 |

                 |

        +-------------------+

        |     Alerting      |

        | Slack             |

        | Email             |

        | PagerDuty         |

        +-------------------+
```

Stack phổ biến:

```text
Metrics:

Prometheus
+
Grafana
+
Alertmanager


Logs:

Fluent Bit
+
Loki
+
Grafana


Enterprise:

Fluent Bit
+
Elasticsearch
+
Kibana
```

Khi kết hợp:

```text
Kubernetes

+

Monitoring

+

Logging

+

Alerting
```

ta có một hệ thống có khả năng:

* Phát hiện lỗi sớm.
* Điều tra nguyên nhân nhanh.
* Theo dõi performance.
* Scale chính xác.
* Vận hành production ổn định.

Đây là nền tảng quan trọng trước khi đi sâu vào các chủ đề nâng cao như:

```text
Prometheus Operator

ServiceMonitor

OpenTelemetry

Distributed Tracing

Jaeger

Grafana Tempo

SLO / SLA / SLI

Production Incident Management
```

