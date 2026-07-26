# 13. Kafka, MinIO và Airflow trên Kubernetes

Ở bài trước, chúng ta đã tìm hiểu về **Apache Spark on Kubernetes**.

Chúng ta biết rằng:

* Spark dùng để xử lý dữ liệu lớn.
* Driver điều khiển job.
* Executor xử lý dữ liệu song song.
* Kubernetes chịu trách nhiệm cấp phát tài nguyên và quản lý lifecycle của Spark workload.

Tuy nhiên, trong một hệ thống Data Engineering thực tế, Spark không hoạt động một mình.

Một Data Platform hoàn chỉnh thường có nhiều thành phần:

```text
                Data Platform Architecture


        Data Source

            |

            v

        Kafka

            |

            v

        Data Lake

            |

            v

        Spark Processing

            |

            v

        Warehouse


            |

            v

        BI / Analytics
```

Ngoài ra cần một công cụ để điều phối pipeline:

```text
Airflow
```

Một kiến trúc phổ biến:

```text
                    Kubernetes Cluster


                            |

        +-------------------+-------------------+

        |                   |                   |

      Kafka              MinIO              Airflow


        |                   |                   |

        |                   |                   |

 Event Streaming       Data Lake          Workflow


                            |

                            |

                          Spark


                            |

                            |

                       Data Processing

```

---

Trong bài này chúng ta sẽ tìm hiểu:

```text
Kafka on Kubernetes

MinIO on Kubernetes

Airflow on Kubernetes

Cách các thành phần kết hợp thành Data Platform
```

---

# 1. Tổng quan kiến trúc Data Platform hiện đại

Một hệ thống dữ liệu thường có 5 lớp:

```text
+------------------------------------------------+

                Data Source Layer


 Database

 API

 IoT

 Application


+------------------------------------------------+

                Streaming Layer


 Kafka


+------------------------------------------------+

                Storage Layer


 MinIO / S3

 Data Lake


+------------------------------------------------+

                Processing Layer


 Spark


+------------------------------------------------+

                Orchestration Layer


 Airflow


+------------------------------------------------+

                Monitoring Layer


 Prometheus

 Grafana

 Logging


+------------------------------------------------+
```

---

Mỗi thành phần có vai trò khác nhau:

| Component  | Vai trò                             |
| ---------- | ----------------------------------- |
| Kafka      | Thu thập và truyền dữ liệu realtime |
| MinIO      | Lưu trữ dữ liệu dạng object storage |
| Spark      | Xử lý dữ liệu lớn                   |
| Airflow    | Điều phối workflow                  |
| Kubernetes | Quản lý toàn bộ workload            |

---

# 2. Vì sao chạy Kafka, MinIO, Airflow trên Kubernetes?

Trước đây:

Kafka thường chạy:

```text
VM riêng

Bare metal server

Hadoop cluster
```

MinIO:

```text
Server vật lý

VM storage cluster
```

Airflow:

```text
VM chạy scheduler + worker
```

---

Nhưng khi dùng Kubernetes:

```text
Một nền tảng duy nhất quản lý tất cả
```

Ví dụ:

```text
Kubernetes Cluster


Namespace data


 |
 +-- Kafka

 |
 +-- MinIO

 |
 +-- Spark

 |
 +-- Airflow
```

---

Lợi ích:

## 2.1 Resource Management

Kubernetes quản lý:

```text
CPU

Memory

Storage

Network
```

Ví dụ:

Kafka broker:

```yaml
resources:
 requests:
   memory: 8Gi
   cpu: 2
```

---

## 2.2 Self Healing

Nếu Kafka broker chết:

```text
Kafka Pod

        X
```

Kubernetes:

```text
Detect failure

      |

Create new Pod

      |

Attach storage

      |

Recover
```

---

## 2.3 Scaling

Ví dụ Kafka:

Ban đầu:

```text
3 brokers
```

Traffic tăng:

```text
6 brokers
```

Kubernetes hỗ trợ mở rộng workload.

---

## 2.4 Infrastructure as Code

Toàn bộ hệ thống mô tả bằng YAML:

```yaml
Kafka

MinIO

Airflow

Spark
```

Có thể:

```bash
git commit

CI/CD deploy

rollback
```

---

# 3. Kafka là gì?

## 3.1 Tổng quan Kafka

Apache Kafka là một distributed event streaming platform.

Kafka dùng để:

```text
Collect events

Store events

Process events

Distribute events
```

Ví dụ:

Một ứng dụng ecommerce:

```text
User click product

        |

        v

     Kafka

        |

        +---- Recommendation

        |

        +---- Analytics

        |

        +---- Monitoring
```

---

# 3.2 Kafka giải quyết vấn đề gì?

Không dùng Kafka:

```text
Application A

        |

        |

Application B
```

Hai hệ thống phụ thuộc trực tiếp.

Nếu B chết:

```text
A bị ảnh hưởng
```

---

Dùng Kafka:

```text
Producer

    |

    v

 Kafka

    |

    +------ Consumer A

    |

    +------ Consumer B
```

Kafka đóng vai trò buffer trung gian.

---

# 4. Các khái niệm Kafka quan trọng

## 4.1 Producer

Producer là ứng dụng gửi message vào Kafka.

Ví dụ:

```python
producer.send(
    "user-events",
    {
      "user":123,
      "action":"login"
    }
)
```

---

## 4.2 Consumer

Consumer đọc dữ liệu từ Kafka.

Ví dụ:

```text
Kafka Topic

       |

       v

Spark Streaming

       |

       v

Data Lake
```

---

## 4.3 Topic

Topic là nơi lưu message.

Ví dụ:

```text
Kafka


Topic:

user-events

payment-events

click-events
```

---

Ví dụ:

Topic:

```text
user-events
```

chứa:

```json
{
"user":1,
"event":"login"
}


{
"user":2,
"event":"purchase"
}
```

---

## 4.4 Partition

Topic được chia thành partition.

Ví dụ:

```text
Topic user-events


Partition 0

Partition 1

Partition 2
```

---

Partition giúp:

```text
Parallel processing

Higher throughput

Scalability
```

---

Ví dụ:

Không partition:

```text
Consumer 1

 đọc toàn bộ
```

Có partition:

```text
Consumer 1 -> partition 0

Consumer 2 -> partition 1

Consumer 3 -> partition 2
```

---

# 5. Kafka Broker là gì?

Broker là server chạy Kafka.

Ví dụ:

```text
Kafka Cluster


Broker 1

Broker 2

Broker 3
```

Mỗi broker lưu một phần dữ liệu.

---

Ví dụ:

```text
Topic payment


Partition 0

   Broker 1


Partition 1

   Broker 2


Partition 2

   Broker 3
```

---

# 6. Kafka Replication

Kafka cần đảm bảo dữ liệu không mất khi broker chết.

Ví dụ:

Replication factor:

```text
3
```

Có nghĩa:

Mỗi partition có 3 bản copy.

Ví dụ:

```text
Partition 0


Leader

 |

+---------+

Replica 1

Replica 2
```

---

Nếu broker leader chết:

```text
Broker 1

   X
```

Kafka chọn replica khác:

```text
Replica 1

become leader
```

---

# 7. Kafka trên Kubernetes Architecture

Kafka là workload stateful.

Không nên chạy Kafka bằng Deployment.

Vì:

Deployment:

```text
Pod không cố định

IP thay đổi

Storage không đảm bảo
```

Kafka cần:

```text
Stable identity

Stable storage

Stable network
```

Do đó dùng:

```text
StatefulSet
```

---

Kiến trúc:

```text
              Kafka Cluster


        StatefulSet kafka


             |

 +-----------+-----------+


 kafka-0    kafka-1    kafka-2


    |          |          |


 PVC        PVC        PVC


    |          |          |


Storage    Storage    Storage
```

---

Mỗi broker có:

```text
Hostname cố định

Volume riêng

Identity riêng
```

---

Ví dụ:

Kafka broker:

```text
kafka-0.kafka-headless.data.svc.cluster.local

kafka-1.kafka-headless.data.svc.cluster.local

kafka-2.kafka-headless.data.svc.cluster.local
```

---

# 8. Kafka cần Headless Service vì sao?

Kafka broker cần biết nhau.

Ví dụ:

```text
Broker 1

cần biết

Broker 2

Broker 3
```

Không thể dùng:

```text
Service ClusterIP
```

vì:

```text
ClusterIP che giấu Pod phía sau
```

Kafka cần từng broker cụ thể.

---

Do đó:

```yaml
clusterIP: None
```

Ví dụ:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: kafka-headless


spec:

  clusterIP: None

  selector:
    app: kafka
```

---

DNS:

```text
kafka-0.kafka-headless.namespace.svc.cluster.local
```

---

# 9. Kafka Storage trên Kubernetes

Kafka lưu:

```text
Messages

Logs

Offsets

Metadata
```

Không thể dùng:

```text
emptyDir
```

vì:

Pod restart:

```text
Data mất
```

---

Kafka cần:

```text
PersistentVolume

PersistentVolumeClaim
```

---

Ví dụ:

```text
Kafka Pod


     |

     |

PVC kafka-data


     |

     |

Persistent Disk
```

---

# 10. Kafka Deployment thực tế

Production thường dùng:

```text
Strimzi Kafka Operator
```

Thay vì tự viết StatefulSet.

---

Strimzi cung cấp:

```text
Kafka CRD

KafkaUser

KafkaTopic

KafkaConnect
```

Ví dụ:

```yaml
apiVersion: kafka.strimzi.io/v1beta2

kind: Kafka


metadata:

 name: kafka-cluster


spec:

 kafka:

  replicas: 3
```

---

Operator tự tạo:

```text
StatefulSet

Service

ConfigMap

PVC

Pod
```

---

# 11. MinIO là gì?

## 11.1 Tổng quan

MinIO là một object storage tương thích với Amazon S3 API.

Dùng để lưu:

```text
Images

Parquet

CSV

ML Model

Backup

Log
```

---

Ví dụ Data Lake:

```text
                MinIO


              bucket:


              raw-data

              processed-data

              models

              backup
```

---

# 12. Object Storage khác File System thế nào?

File system:

```text
/home/user/data.csv
```

Có:

```text
Folder

File hierarchy
```

---

Object storage:

```text
bucket/object
```

Ví dụ:

```text
s3://lake/raw/events/2026/data.parquet
```

---

Object storage phù hợp:

```text
Big Data

Data Lake

AI/ML
```

---

# 13. MinIO Architecture trên Kubernetes

MinIO cũng là stateful workload.

Kiến trúc:

```text
              MinIO Cluster


        StatefulSet


             |


 +-----------+-----------+


 minio-0   minio-1   minio-2


    |         |         |


 PVC       PVC       PVC
```

---

MinIO distributed mode:

```text
Multiple disks

Multiple nodes

Erasure coding
```

---

# 14. MinIO Service

Có hai Service thường gặp:

## API Service

Dùng cho application:

```text
Port 9000
```

Ví dụ:

```text
Spark

 |

s3a://bucket

 |

MinIO API
```

---

## Console Service

UI quản lý:

```text
Port 9001
```

Dùng để:

```text
Create bucket

Upload file

Manage user
```

---

# 15. MinIO với Spark

Spark đọc:

```text
s3a://lakehouse/data
```

MinIO cung cấp:

```text
S3 API
```

Luồng:

```text
Spark Executor


      |

      |

 Hadoop S3A Connector


      |

      |

 MinIO Service


      |

      |

 Parquet files
```

---

# 16. Airflow là gì?

Apache Airflow là workflow orchestration platform.

Nó dùng để:

```text
Schedule jobs

Manage dependencies

Monitor pipelines
```

---

Ví dụ ETL:

```text
01 Extract data

        |

02 Transform Spark

        |

03 Load Warehouse

        |

04 Send report
```

Airflow quản lý thứ tự này.

---

# 17. Airflow Architecture

Airflow gồm:

```text
Webserver

Scheduler

Worker

Metadata Database

Executor
```

---

Kiến trúc:

```text
                 Airflow


              Webserver


                  |


              Scheduler


                  |


        +---------+---------+


        |                   |


      Worker             Worker


                  |


            Metadata DB

```

---

# 18. Airflow Components

## Webserver

UI:

```text
View DAG

Trigger DAG

Check logs
```

---

## Scheduler

Quan trọng nhất.

Nó:

```text
Check schedule

Create tasks

Send tasks to executor
```

---

## Worker

Chạy task.

Ví dụ:

```python
spark-submit job.py
```

---

## Metadata Database

Lưu:

```text
DAG status

Task status

History
```

Thường dùng:

```text
PostgreSQL

MySQL
```

---

# 19. Airflow trên Kubernetes

Airflow có thể chạy:

```text
Docker

VM

Kubernetes
```

Trên Kubernetes:

```text
Airflow Pods
```

---

Kiến trúc:

```text
Namespace airflow


 +-- airflow-webserver

 +-- airflow-scheduler

 +-- airflow-worker

 +-- postgres
```

---

# 20. Airflow Executor

Airflow có nhiều executor.

## SequentialExecutor

Chạy tuần tự.

Dùng:

```text
Development
```

---

## LocalExecutor

Chạy nhiều process trên một máy.

---

## CeleryExecutor

Worker cluster.

---

## KubernetesExecutor

Mỗi task tạo một Pod.

Đây là cách cloud native nhất.

---

Ví dụ:

DAG:

```text
Task A

Task B

Task C
```

Kubernetes tạo:

```text
task-a-pod

task-b-pod

task-c-pod
```

Task xong:

```text
Pod delete
```

---
# 21. Airflow KubernetesExecutor

Trong Kubernetes, một trong những điểm mạnh nhất của Airflow là khả năng chạy mỗi task như một Kubernetes Pod riêng biệt.

Cách này gọi là:

```text
KubernetesExecutor
```

---

## 21.1 Vấn đề của Worker cố định

Với CeleryExecutor truyền thống:

```text
Airflow Scheduler

        |

        |

     Worker 1

     Worker 2

     Worker 3
```

Worker phải luôn chạy.

Ví dụ:

```text
Ban ngày:

100 tasks

cần:

10 workers
```

Nhưng ban đêm:

```text
5 tasks

cần:

1 worker
```

Vấn đề:

```text
Worker vẫn tồn tại

Resource vẫn bị chiếm dụng
```

---

## 21.2 KubernetesExecutor giải quyết như thế nào?

Với KubernetesExecutor:

```text
Airflow Scheduler

        |

        |

 Kubernetes API

        |

        |

Create Pod chạy task
```

Ví dụ:

DAG:

```text
extract_data

transform_data

load_data
```

Airflow tạo:

```text
extract-data-pod

transform-data-pod

load-data-pod
```

Sau khi task hoàn thành:

```text
Pod terminated
```

---

Kiến trúc:

```text
                 Kubernetes Cluster


                     Airflow


                  Scheduler


                       |

                       |

              Kubernetes API Server


                       |

        +--------------+--------------+

        |              |              |

     Task Pod      Task Pod       Task Pod


```

---

## 21.3 Lợi ích KubernetesExecutor

### Dynamic Scaling

Không cần đoán trước số worker.

Ví dụ:

```text
1000 tasks

      |

1000 Pods được tạo
```

Sau khi xong:

```text
1000 Pods biến mất
```

---

### Isolation

Mỗi task có environment riêng:

Ví dụ:

Task A:

```yaml
image: python:3.11
```

Task B:

```yaml
image: spark:3.5
```

Không bị conflict dependency.

---

### Resource Control

Mỗi task có resource riêng:

Ví dụ:

```yaml
resources:
 requests:
   cpu: 2
   memory: 4Gi
```

Task nặng:

```text
4 CPU

8GB RAM
```

Task nhẹ:

```text
100m CPU

256MB RAM
```

---

# 22. Helm Chart cho Airflow

Trong thực tế không deploy Airflow bằng YAML thủ công.

Thường dùng:

```text
Helm Chart
```

Ví dụ:

```text
Apache Airflow Helm Chart
```

---

Cấu trúc:

```text
airflow-chart


values.yaml


templates/


    deployment.yaml

    service.yaml

    configmap.yaml

    secret.yaml
```

---

Cài Airflow:

```bash
helm repo add apache-airflow https://airflow.apache.org

helm install airflow apache-airflow/airflow \
-n airflow \
--create-namespace
```

---

Helm sẽ tạo:

```text
Namespace airflow


Deployment:

airflow-webserver

airflow-scheduler


StatefulSet:

postgres


Service:

airflow-webserver


Secret:

database password
```

---

# 23. Data Platform hoàn chỉnh trên Kubernetes

Bây giờ ghép tất cả thành một hệ thống.

Kiến trúc:

```text
                         Kubernetes Cluster


                                 |

        +------------------------+-------------------------+

        |                        |                         |

      Kafka                    Airflow                  MinIO


        |                        |                         |

        |                        |                         |

  Streaming Data          Workflow Control          Data Lake


        |                        |

        |                        |

        +------------------------+

                    |

                    |

                  Spark


                    |

                    |

              Process Data


                    |

                    |

              Analytics System

```

---

# 24. Luồng dữ liệu thực tế

Ví dụ hệ thống ecommerce.

---

## Bước 1: Application gửi event

User mua hàng:

```json
{
"user_id":1001,
"product":"iphone",
"price":999
}
```

Application gửi:

```text
Kafka Topic:

purchase-events
```

---

## Bước 2: Kafka lưu event

Kafka:

```text
purchase-events


Partition 0

Partition 1

Partition 2
```

---

## Bước 3: Spark đọc Kafka

Spark Structured Streaming:

```python
spark.readStream \
.format("kafka") \
.option(
"kafka.bootstrap.servers",
"kafka:9092"
)
```

---

Spark xử lý:

```text
Raw Event

     |

Cleaning

     |

Aggregation

     |

Transformation
```

---

## Bước 4: Lưu vào MinIO

Spark ghi:

```text
s3a://lakehouse/purchase/
```

Ví dụ:

```text
lakehouse

 |
 +-- raw

 |
 +-- processed

 |
 +-- warehouse
```

---

## Bước 5: Airflow điều phối

Airflow DAG:

```text
extract_task

      |

spark_job

      |

validate_data

      |

publish_report
```

---

# 25. Namespace thiết kế cho Data Platform

Trong production không nên để tất cả chung namespace.

Ví dụ:

```text
kubernetes cluster


namespace:

data-streaming

    |
    + Kafka


data-storage

    |
    + MinIO


data-processing

    |
    + Spark


workflow

    |
    + Airflow


monitoring

    |
    + Prometheus
```

---

Lợi ích:

## Resource Isolation

Ví dụ:

Kafka:

```yaml
requests:
 cpu: 4
 memory: 16Gi
```

Spark:

```yaml
requests:
 cpu: 8
 memory: 32Gi
```

Không tranh chấp tài nguyên.

---

## RBAC

Team Kafka:

```text
chỉ được quản lý namespace kafka
```

Team Spark:

```text
chỉ được submit Spark job
```

---

## Quota

Ví dụ:

Namespace Spark:

```yaml
apiVersion: v1
kind: ResourceQuota

spec:

 hard:

  requests.cpu: "20"

  requests.memory: "64Gi"
```

---

# 26. Storage Architecture cho Data Platform

Một vấn đề rất quan trọng:

Kafka, MinIO, Database đều cần storage.

Không được dùng:

```text
emptyDir
```

vì:

```text
Pod restart

Data mất
```

---

Kiến trúc:

```text
                 Kubernetes


                    Pod


                     |

                     |

                    PVC


                     |

                     |

                    PV


                     |

                     |

              Storage Backend

```

---

Ví dụ:

MinIO:

```text
minio-0

 |

PVC-minio-0

 |

Disk
```

Kafka:

```text
kafka-0

 |

PVC-kafka-0

 |

Disk
```

---

# 27. Database trên Kubernetes

Trong Data Platform thường có database:

Ví dụ:

```text
Airflow Metadata DB

PostgreSQL

MySQL

Redis
```

---

Database cũng là stateful workload.

Không dùng:

```text
Deployment
```

mà dùng:

```text
StatefulSet
```

---

Ví dụ:

```text
postgres-0


postgres-1


postgres-2
```

Mỗi instance:

```text
PVC riêng
```

---

Tuy nhiên production thường dùng:

Managed Database:

```text
AWS RDS

Cloud SQL

Azure Database
```

thay vì tự quản lý database trong Kubernetes.

---

# 28. Secret Management

Data Platform chứa rất nhiều credential:

Ví dụ:

Kafka:

```text
username

password
```

MinIO:

```text
access key

secret key
```

Database:

```text
postgres password
```

---

Không được hard-code:

Sai:

```yaml
env:

- name: PASSWORD

  value: admin123
```

---

Dùng Kubernetes Secret:

```yaml
apiVersion: v1

kind: Secret

metadata:

 name: minio-secret


data:

 accesskey: xxxx

 secretkey: yyyy
```

---

Pod mount:

```yaml
envFrom:

- secretRef:

    name: minio-secret
```

---

# 29. Networking giữa Kafka, MinIO, Airflow, Spark

Ví dụ namespace:

```text
data
```

Kafka Service:

```text
kafka.data.svc.cluster.local
```

MinIO:

```text
minio.data.svc.cluster.local
```

Airflow:

```text
airflow.data.svc.cluster.local
```

---

Spark config:

```python
spark.hadoop.fs.s3a.endpoint=

http://minio.data.svc.cluster.local:9000
```

---

Kafka connection:

```text
bootstrap-server:

kafka.data.svc.cluster.local:9092
```

---

Airflow connection:

```text
spark://spark.data.svc.cluster.local
```

---

# 30. Deployment Strategy trong Data Platform

Không phải component nào cũng deploy giống nhau.

---

## Stateless

Ví dụ:

```text
Airflow Webserver

API Gateway

Frontend
```

Dùng:

```text
Deployment
```

---

## Stateful

Ví dụ:

```text
Kafka

MinIO

PostgreSQL

Redis
```

Dùng:

```text
StatefulSet
```

---

## Batch Job

Ví dụ:

```text
Spark ETL

Migration

Data Cleanup
```

Dùng:

```text
Job

CronJob

Spark Operator
```

---

# 31. Monitoring Data Platform

Một hệ thống Data Platform cần monitoring tất cả component.

---

## Kafka Monitoring

Theo dõi:

```text
Broker status

Consumer lag

Throughput

Partition health
```

---

## MinIO Monitoring

Theo dõi:

```text
Disk usage

Object count

Request latency
```

---

## Airflow Monitoring

Theo dõi:

```text
DAG success rate

Task duration

Failed tasks
```

---

## Spark Monitoring

Theo dõi:

```text
Executor memory

GC time

Stage duration
```

---

Stack phổ biến:

```text
Prometheus

+

Grafana

+

Loki

+

AlertManager
```

---

# 32. Logging Architecture

Trong Kubernetes:

Pod sinh log:

```text
stdout

stderr
```

Ví dụ:

```bash
kubectl logs kafka-0
```

---

Production:

Không xem log thủ công.

Dùng:

```text
Fluent Bit

+

Loki

+

Grafana
```

---

Luồng:

```text
Kafka Pod


    |

    |

Fluent Bit


    |

    |

Loki


    |

    |

Grafana
```

---

# 33. Backup và Disaster Recovery

Stateful system cần backup.

---

## Kafka

Backup:

```text
Topic metadata

Messages

Configuration
```

---

## MinIO

Backup:

```text
Bucket

Object

Metadata
```

---

## Airflow

Backup:

```text
DAG

Metadata Database

Connection
```

---

Ví dụ:

MinIO replication:

```text
Cluster A

      |

      |

Cluster B
```

---

# 34. Production Best Practices

## Kafka

Nên:

```text
3+ brokers

Replication >=3

Persistent storage

Monitoring consumer lag
```

---

## MinIO

Nên:

```text
Distributed mode

Multiple disks

TLS enabled

Backup
```

---

## Airflow

Nên:

```text
KubernetesExecutor

External PostgreSQL

Git-based DAG deployment

Secret management
```

---

## Kubernetes

Nên:

```text
Resource requests/limits

Namespace isolation

RBAC

Monitoring

NetworkPolicy
```

---

# 35. Troubleshooting Data Platform

---

# Kafka Pod không start

Kiểm tra:

```bash
kubectl describe pod kafka-0
```

Các lỗi thường gặp:

```text
PVC Pending

Storage thiếu

Memory limit thấp

Wrong broker configuration
```

---

# Kafka consumer lag cao

Kiểm tra:

```text
Consumer đang chậm hơn producer
```

Nguyên nhân:

```text
Spark job chậm

Partition ít

Consumer thiếu resource
```

---

# MinIO không connect được từ Spark

Kiểm tra:

```bash
kubectl get svc -n minio
```

Test:

```bash
curl http://minio:9000
```

Kiểm tra:

```text
Endpoint đúng

Access key đúng

S3A connector có tồn tại
```

---

# Airflow task fail

Kiểm tra:

```bash
kubectl logs <task-pod>
```

Nguyên nhân:

```text
Permission

Image lỗi

Resource thiếu

Dependency lỗi
```

---

# Spark không đọc được MinIO

Kiểm tra:

```text
hadoop-aws jar

AWS SDK jar

S3A configuration
```

Ví dụ:

```python
spark.hadoop.fs.s3a.impl

org.apache.hadoop.fs.s3a.S3AFileSystem
```

---

# 36. Bài lab thực hành

## Lab 1: Deploy MinIO

Yêu cầu:

```text
Namespace minio

StatefulSet

PVC

Service

```

Test:

```bash
mc alias set

mc ls
```

---

## Lab 2: Deploy Kafka

Yêu cầu:

```text
Kafka cluster 3 brokers

Headless Service

Persistent Storage
```

Test:

```bash
kafka-topics.sh --list
```

---

## Lab 3: Deploy Airflow

Yêu cầu:

```text
Airflow scheduler

Webserver

PostgreSQL
```

Test:

```text
Create DAG

Run DAG
```

---

## Lab 4: End-to-end pipeline

Tạo flow:

```text
Producer

   |

Kafka

   |

Spark Streaming

   |

MinIO

   |

Airflow Schedule
```

---

# 37. Checklist cần nắm sau bài này

Sau bài này cần hiểu:

```text
Kafka dùng để làm gì?

Topic, Partition, Broker là gì?

Vì sao Kafka cần StatefulSet?

Vì sao Kafka cần Headless Service?

MinIO khác filesystem như thế nào?

Vì sao MinIO phù hợp Data Lake?

Airflow gồm những component nào?

Scheduler làm gì?

KubernetesExecutor hoạt động thế nào?

Khi nào dùng Deployment?

Khi nào dùng StatefulSet?

Kafka, MinIO, Airflow giao tiếp qua Service DNS như thế nào?

Data Platform trên Kubernetes có kiến trúc ra sao?
```

---

# 38. Tổng kết

Một Data Platform hiện đại trên Kubernetes thường có kiến trúc:

```text
                         Kubernetes


                              |

        +---------------------+--------------------+

        |                     |                    |

      Kafka                MinIO              Airflow


        |                     |                    |

 Streaming             Data Lake            Workflow


        |

        |

      Spark


        |

        |

 Analytics / Warehouse

```

Vai trò:

```text
Kafka
=
Thu thập và truyền dữ liệu realtime


MinIO
=
Lưu trữ dữ liệu dạng object storage


Spark
=
Xử lý dữ liệu lớn


Airflow
=
Điều phối pipeline


Kubernetes
=
Quản lý toàn bộ lifecycle, resource, networking, storage
```

Hiểu được cách triển khai Kafka + MinIO + Airflow trên Kubernetes là nền tảng quan trọng để xây dựng các hệ thống:

```text
Data Lake

Lakehouse

ML Platform

Streaming Platform

Enterprise Data Platform
```

