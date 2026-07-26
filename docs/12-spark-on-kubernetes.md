# 12. Apache Spark on Kubernetes: Chạy Big Data Processing trên Kubernetes

Ở các bài trước, chúng ta đã tìm hiểu Kubernetes quản lý workload như thế nào:

* **Pod** chạy container.
* **Deployment** quản lý ứng dụng stateless.
* **StatefulSet** quản lý ứng dụng stateful.
* **Service** giúp các component tìm thấy nhau.
* **Storage** cung cấp dữ liệu bền vững.
* **RBAC** kiểm soát quyền truy cập.
* **Monitoring + Logging** giúp vận hành production.

Tuy nhiên, Kubernetes ban đầu được thiết kế chủ yếu để chạy các ứng dụng container như:

```text
Web API
Backend service
Frontend
Microservice
Database cluster
```

Trong thực tế Data Engineering còn có một nhóm workload rất quan trọng:

```text
Batch Processing

ETL Pipeline

Data Transformation

Machine Learning Training

Data Analytics
```

Ví dụ:

```text
Đọc 10TB dữ liệu từ Data Lake

       |

Transform bằng Spark

       |

Ghi kết quả về Data Warehouse
```

Đây chính là nơi **Apache Spark on Kubernetes** xuất hiện.

---

# 1. Spark là gì?

## 1.1 Tổng quan Apache Spark

Apache Spark là một distributed computing framework dùng để xử lý dữ liệu lớn.

Spark có khả năng:

```text
Xử lý hàng GB -> TB -> PB dữ liệu

Chạy song song trên nhiều máy

In-memory processing

Fault tolerance

Scale ngang
```

Spark thường được dùng cho:

```text
ETL

Data Lake processing

Machine Learning

Streaming

Analytics
```

---

Ví dụ một pipeline Data Engineering:

```text
                Data Source

                    |

              Kafka / S3 / MinIO

                    |

                    v

              Apache Spark Job

                    |

        +-----------+------------+

        |                        |

 Transformation             Aggregation


                    |

                    v

             Data Warehouse
```

---

# 2. Vì sao chạy Spark trên Kubernetes?

Trước đây Spark thường chạy trên:

```text
Standalone Cluster

YARN (Hadoop)

Mesos
```

Ví dụ Hadoop:

```text
              Hadoop Cluster


        ResourceManager

              |

        NodeManager

     +--------+--------+

     |        |        |

  Worker   Worker   Worker
```

---

Nhưng hiện nay nhiều công ty đã dùng Kubernetes làm nền tảng chung:

```text
                 Kubernetes Cluster


        +--------------------------+

        |                          |

     Web App                 Spark Job


        |                          |

        +--------------------------+

                 Nodes
```

Thay vì phải quản lý:

```text
Một cluster cho app

Một cluster cho Spark

Một cluster cho ML
```

Có thể dùng chung:

```text
Một Kubernetes Cluster
```

---

# 3. Lợi ích của Spark trên Kubernetes

## 3.1 Unified Infrastructure

Không cần:

```text
Kubernetes cluster

+

Spark standalone cluster
```

Chỉ cần:

```text
Kubernetes cluster
```

Ví dụ:

```text
Namespace production

    |
    +-- Backend API

    +-- Kafka

    +-- Spark Jobs

    +-- Airflow

    +-- MinIO
```

---

## 3.2 Dynamic Resource Allocation

Spark workload thường không chạy liên tục.

Ví dụ:

Ban ngày:

```text
Spark job chạy ETL

100 executor
```

Ban đêm:

```text
Không chạy

0 executor
```

Kubernetes có thể:

```text
Tạo Pod khi cần

Xóa Pod khi xong
```

Tiết kiệm tài nguyên.

---

## 3.3 Isolation bằng Namespace

Ví dụ:

```text
Namespace data


Spark Jobs


Namespace application


Backend API
```

Có thể áp dụng:

```text
ResourceQuota

RBAC

NetworkPolicy
```

---

## 3.4 Kubernetes Scheduler tốt hơn

Kubernetes hiểu:

```text
CPU

Memory

GPU

Node label

Taints

Affinity
```

Ví dụ:

Spark ML job cần GPU:

```yaml
nodeSelector:

  gpu: true
```

Kubernetes tự tìm node phù hợp.

---

# 4. Kiến trúc Spark trên Kubernetes

Spark trên Kubernetes có kiến trúc:

```text
                 Spark Application


                       |

                       v


                  Spark Driver


                       |

        +--------------+--------------+

        |                             |

        v                             v


   Executor Pod                 Executor Pod


        |                             |


        +-------------+---------------+

                      |

                      v

                Kubernetes API
```

Có hai loại Pod chính:

```text
Driver Pod

Executor Pod
```

---

# 5. Spark Driver là gì?

Driver là thành phần điều khiển Spark application.

Nó chịu trách nhiệm:

```text
Create SparkContext

Schedule task

Track executor

Coordinate jobs

Maintain metadata
```

Có thể hiểu:

```text
Driver = bộ não của Spark job
```

---

Ví dụ:

Ta chạy:

```python
df = spark.read.parquet(
    "s3a://lake/data"
)

df.groupBy("country").count()

df.write.parquet(
    "s3a://lake/result"
)
```

Driver sẽ:

```text
1.

Đọc logical plan


2.

Chia thành stages


3.

Gửi tasks cho executor


4.

Collect result
```

---

# 6. Executor là gì?

Executor là worker thực sự xử lý dữ liệu.

Ví dụ:

```text
Driver


 |
 |
 +---- Executor 1

 |
 +---- Executor 2

 |
 +---- Executor 3
```

Executor thực hiện:

```text
Read partition

Transform data

Shuffle

Write output
```

---

Ví dụ:

Dataset:

```text
10TB data
```

Spark chia:

```text
Partition 1
Partition 2
Partition 3
...
Partition 1000
```

Executor xử lý song song:

```text
Executor 1:

partition 1-100


Executor 2:

partition 101-200
```

---

# 7. Spark Application lifecycle trên Kubernetes

Khi submit Spark job:

```bash
spark-submit \
--master k8s://cluster \
app.py
```

Luồng:

---

## Bước 1: Submit application

User chạy:

```bash
spark-submit
```

---

## Bước 2: Kubernetes tạo Driver Pod

Ví dụ:

```text
spark-pi-driver
```

---

## Bước 3: Driver request Executor

Driver gọi Kubernetes API:

```text
Create executor pod
```

---

## Bước 4: Executor được tạo

Ví dụ:

```text
spark-pi-exec-1

spark-pi-exec-2

spark-pi-exec-3
```

---

## Bước 5: Spark chạy computation

```text
Driver

   |

   +--- Executor

   +--- Executor

   +--- Executor
```

---

## Bước 6: Job hoàn thành

Executor Pod bị xóa.

Driver Pod kết thúc.

---

# 8. Spark Operator trên Kubernetes

Có hai cách chạy Spark:

```text
spark-submit

Spark Operator
```

---

# 8.1 spark-submit

Giống Spark truyền thống.

Ví dụ:

```bash
spark-submit \
--master k8s://https://kubernetes.default.svc \
--deploy-mode cluster \
app.py
```

---

Ưu điểm:

```text
Đơn giản

Giống Spark cũ
```

Nhược điểm:

```text
Khó quản lý nhiều job

Khó integrate Kubernetes workflow
```

---

# 8.2 Spark Operator

Spark Operator là Kubernetes controller quản lý Spark application.

Nó tạo CRD:

```text
SparkApplication
```

Thay vì:

```bash
spark-submit
```

Ta tạo YAML:

```yaml
apiVersion: sparkoperator.k8s.io/v1beta2
kind: SparkApplication

metadata:
  name: spark-job

spec:

  type: Python

  mode: cluster

  image:
    spark-image:latest

  driver:
    cores: 1
    memory: 2g

  executor:

    instances: 3
    cores: 2
    memory: 4g
```

Apply:

```bash
kubectl apply -f spark-job.yaml
```

Kubernetes tự:

```text
Create driver

Create executor

Monitor job

Restart failed job
```

---

# 9. Spark Namespace

Best practice:

Không chạy Spark ở namespace default.

Ví dụ:

```text
hr-spark
```

Tạo:

```bash
kubectl create namespace hr-spark
```

Các resource:

```text
Namespace hr-spark


 |
 +-- Spark Driver Pod

 |
 +-- Executor Pods

 |
 +-- ServiceAccount

 |
 +-- ConfigMap
```

---

# 10. RBAC cho Spark

Spark cần quyền tạo Executor Pod.

Vì Driver phải gọi Kubernetes API.

Luồng:

```text
Spark Driver

      |

      |

Kubernetes API

      |

Create Executor Pod
```

Do đó cần:

```text
ServiceAccount

Role

RoleBinding
```

---

Ví dụ:

ServiceAccount:

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: spark

  namespace: hr-spark
```

---

Role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1

kind: Role

metadata:

  name: spark-role

  namespace: hr-spark


rules:

- apiGroups:
  - ""

  resources:

  - pods

  - services

  verbs:

  - create

  - get

  - list

  - delete
```

---

Binding:

```yaml
kind: RoleBinding

metadata:

  name: spark-rolebinding


subjects:

- kind: ServiceAccount

  name: spark


roleRef:

  kind: Role

  name: spark-role
```

---

# 11. Spark + Object Storage (MinIO / S3)

Trong Data Engineering, Spark thường đọc dữ liệu từ:

```text
S3

MinIO

HDFS

Azure Blob
```

Ví dụ:

```text
Spark

 |

 |

s3a://lakehouse/data

 |

 |

MinIO
```

---

Spark cần Hadoop AWS connector:

```text
hadoop-aws.jar

aws-java-sdk-bundle.jar
```

---

Config:

```properties
spark.hadoop.fs.s3a.endpoint=http://minio.hr-minio.svc.cluster.local:9000

spark.hadoop.fs.s3a.access.key=minio

spark.hadoop.fs.s3a.secret.key=password

spark.hadoop.fs.s3a.path.style.access=true

spark.hadoop.fs.s3a.connection.ssl.enabled=false
```

---

# 12. Spark đọc Parquet từ MinIO

Ví dụ PySpark:

```python
from pyspark.sql import SparkSession


spark = SparkSession.builder \
    .appName("demo") \
    .getOrCreate()


df = spark.read.parquet(
    "s3a://lakehouse/events/"
)


df.show()
```

---

Luồng:

```text
Executor Pod

      |

      |

S3A Connector

      |

      |

MinIO Service

      |

      |

Parquet file
```

---

# 13. Resource Configuration cho Spark

Spark cần khai báo:

```text
Driver resource

Executor resource
```

---

Ví dụ:

Driver:

```yaml
driver:

  cores: 2

  memory: 4g
```

Executor:

```yaml
executor:

  instances: 3

  cores: 4

  memory: 8g
```

Tổng:

```text
Driver:

2 CPU

4GB RAM


Executors:

3 * 4 CPU

3 * 8GB RAM
```

Cluster cần đủ:

```text
14 CPU

28GB RAM
```

---

# 14. Spark Shuffle

Một khái niệm quan trọng.

Ví dụ:

```python
df.groupBy(
"country"
).count()
```

Spark cần gom dữ liệu cùng key.

Ví dụ:

Executor 1:

```text
Vietnam rows
```

Executor 2:

```text
Vietnam rows
```

Cần gửi về cùng nơi.

Quá trình này gọi:

```text
Shuffle
```

---

Shuffle tốn:

```text
Network

Disk

Memory
```

Trong Kubernetes cần chú ý:

```text
Executor memory

Local disk

Network bandwidth
```

---

# 15. Spark UI trên Kubernetes

Spark Driver expose UI:

```text
4040
```

Có thể xem:

```text
Jobs

Stages

Tasks

Executors

SQL plan
```

---

Port:

```text
4040
```

Ví dụ:

```yaml
spark.ui.port=4040
```

---

Có thể expose bằng:

```text
Service

Ingress

Port-forward
```

Ví dụ:

```bash
kubectl port-forward \
pod/spark-driver \
4040:4040
```

Mở:

```text
http://localhost:4040
```

---

# 16. Troubleshooting Spark on Kubernetes

## 16.1 Executor không tạo được

Kiểm tra:

```bash
kubectl get pods -n hr-spark
```

Nếu:

```text
Pending
```

Kiểm tra:

```bash
kubectl describe pod executor-name
```

Nguyên nhân:

```text
Thiếu CPU

Thiếu memory

Quota limit

Node không đủ resource
```

---

# 16.2 Driver CrashLoopBackOff

Xem:

```bash
kubectl logs driver-pod
```

Các lỗi thường gặp:

```text
Missing jar

Wrong S3 config

Permission denied

Python dependency error
```

---

# 16.3 Không đọc được MinIO

Lỗi:

```text
ClassNotFoundException:

org.apache.hadoop.fs.s3a.S3AFileSystem
```

Nguyên nhân:

Thiếu:

```text
hadoop-aws.jar
```

hoặc:

```text
aws-java-sdk-bundle.jar
```

---

# 16.4 Driver không tạo executor

Kiểm tra RBAC:

```bash
kubectl auth can-i create pods \
--as=system:serviceaccount:hr-spark:spark \
-n hr-spark
```

Nếu:

```text
no
```

RBAC sai.

---

# 17. Best Practices Spark on Kubernetes

## 17.1 Dùng namespace riêng

Ví dụ:

```text
hr-spark
```

---

## 17.2 Dùng ServiceAccount riêng

Không dùng:

```text
default account
```

---

## 17.3 Set resource rõ ràng

Không để:

```text
Unlimited memory
```

---

## 17.4 Dùng Spark Operator

Production nên dùng:

```text
Spark Operator

Airflow

Argo Workflow
```

để orchestration.

---

## 17.5 Monitoring Spark

Kết hợp:

```text
Prometheus

Grafana

Spark metrics
```

Theo dõi:

```text
Executor memory

GC time

Shuffle read/write

Task duration
```

---

# 18. So sánh Spark Standalone vs Spark on Kubernetes

|                 | Spark Standalone | Spark Kubernetes   |
| --------------- | ---------------- | ------------------ |
| Cluster manager | Spark            | Kubernetes         |
| Resource        | Spark quản lý    | Kubernetes quản lý |
| Isolation       | Thấp             | Cao                |
| Namespace       | Không            | Có                 |
| RBAC            | Không            | Có                 |
| Autoscaling     | Hạn chế          | Tốt                |
| Cloud native    | Không            | Có                 |

---

# 19. Tổng kết kiến thức cần nhớ

Sau bài này cần hiểu:

```text
Spark là gì?

Vì sao chạy Spark trên Kubernetes?

Driver là gì?

Executor là gì?

Spark Application lifecycle?

spark-submit khác Spark Operator thế nào?

SparkApplication CRD là gì?

Spark cần RBAC vì sao?

ServiceAccount của Spark dùng làm gì?

Spark đọc dữ liệu từ MinIO/S3 như thế nào?

hadoop-aws.jar dùng để làm gì?

Resource cho Driver và Executor tính thế nào?

Spark UI dùng để debug gì?

Shuffle là gì?

Các lỗi Spark Kubernetes thường gặp?
```

---

# 20. Tổng kết kiến trúc hoàn chỉnh Data Platform trên Kubernetes

Một hệ thống Data Engineering hiện đại:

```text
                         Kubernetes Cluster


                           Ingress


                              |

                              |


        +---------------------+--------------------+


        |                     |                    |


     Airflow              Spark Jobs            Kafka


        |                     |                    |


        |                     |                    |


        +---------------------+--------------------+


                              |


                              |


                         Data Lake


                              |


                         MinIO / S3


                              |


                         Warehouse
```

---

Kubernetes + Spark tạo ra một nền tảng Data Platform cloud-native:

```text
Kubernetes
    +
Spark
    +
MinIO/S3
    +
Airflow
    +
Kafka
    +
Prometheus/Grafana
```

Đây là kiến trúc phổ biến trong các hệ thống:

* Data Lake.
* ETL Platform.
* Machine Learning Platform.
* Big Data Processing Platform.

Sau khi nắm chắc Spark trên Kubernetes, các chủ đề tiếp theo nên học:

```text
13 - Airflow on Kubernetes

14 - Kafka on Kubernetes

15 - Data Lake Architecture

16 - CI/CD Deploy Kubernetes

17 - Production Kubernetes Architecture
```
