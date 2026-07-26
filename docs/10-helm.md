# Helm trong Kubernetes: Quản lý, đóng gói và triển khai ứng dụng Kubernetes chuyên nghiệp

Ở các bài trước, chúng ta đã tìm hiểu Kubernetes từ những thành phần nền tảng:

* Pod, Deployment, ReplicaSet giúp chạy workload.
* Service, DNS giúp các ứng dụng tìm thấy nhau trong cluster.
* ConfigMap, Secret giúp quản lý cấu hình.
* Volume, PVC, StorageClass giúp lưu trữ dữ liệu.
* StatefulSet giúp chạy các hệ thống stateful như Kafka, Database, MinIO.
* RBAC, ServiceAccount giúp kiểm soát quyền truy cập.
* Ingress, LoadBalancer giúp expose ứng dụng ra bên ngoài.
* Healthcheck, Rollout, Rollback giúp vận hành ứng dụng an toàn.

Sau khi học xong những thành phần này, chúng ta có thể triển khai một ứng dụng hoàn chỉnh trên Kubernetes.

Ví dụ một hệ thống production:

```text
Frontend
   |
Ingress
   |
Backend Deployment
   |
Service
   |
Database StatefulSet
   |
PersistentVolume

ConfigMap
Secret
ServiceAccount
RBAC
```

Tuy nhiên, khi hệ thống lớn lên, chúng ta sẽ gặp một vấn đề:

**Kubernetes sử dụng rất nhiều file YAML.**

Một ứng dụng thực tế có thể có:

```text
deployment.yaml
service.yaml
configmap.yaml
secret.yaml
ingress.yaml
serviceaccount.yaml
role.yaml
rolebinding.yaml
pvc.yaml
hpa.yaml
networkpolicy.yaml
```

Chỉ một ứng dụng đơn giản cũng có thể cần hàng chục file YAML.

Nếu công ty có nhiều môi trường:

```text
development
staging
production
```

thì vấn đề càng lớn.

Ví dụ:

Development:

```yaml
replicas: 1
resources:
  memory: 512Mi
```

Production:

```yaml
replicas: 10
resources:
  memory: 8Gi
```

Nếu quản lý bằng YAML thuần, chúng ta phải copy file và sửa thủ công.

Điều này dẫn đến nhiều vấn đề:

```text
Duplicate YAML
Sai cấu hình giữa môi trường
Khó upgrade version
Khó rollback
Khó chia sẻ ứng dụng
Khó đóng gói ứng dụng
```

Đây chính là lý do Helm ra đời.

---

# 1. Helm là gì?

## 1.1 Định nghĩa Helm

**Helm là package manager cho Kubernetes.**

Nói đơn giản:

> Helm giúp đóng gói, cấu hình, triển khai và quản lý ứng dụng Kubernetes giống như cách apt quản lý package trên Ubuntu hoặc npm quản lý package của Node.js.

Ví dụ:

Ubuntu:

```bash
apt install nginx
```

Node.js:

```bash
npm install express
```

Python:

```bash
pip install flask
```

Kubernetes:

```bash
helm install my-app my-chart
```

Thay vì tự tạo từng YAML:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f configmap.yaml
```

Ta có thể:

```bash
helm install backend ./backend-chart
```

Helm sẽ tự:

```text
Render template
        |
        v
Sinh YAML Kubernetes
        |
        v
kubectl apply
        |
        v
Tạo resource
```

---

# 2. Vì sao Kubernetes cần Helm?

Để hiểu Helm, cần hiểu vấn đề của Kubernetes YAML.

---

# 2.1 Kubernetes chỉ hiểu YAML tĩnh

Kubernetes nhận manifest:

Ví dụ Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nginx

spec:
  replicas: 3

  template:
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

Kubernetes không quan tâm:

* Đây là dev hay production.
* Ai deploy.
* Version trước là gì.
* Cấu hình thay đổi thế nào.

Nó chỉ nhận trạng thái mong muốn:

```text
Desired State
```

và cố gắng đưa cluster về trạng thái đó.

---

# 2.2 Vấn đề khi có nhiều môi trường

Giả sử có ứng dụng backend.

## Development

```yaml
replicas: 1

image:
  backend:v1
```

## Production

```yaml
replicas: 10

image:
  backend:v5
```

Nếu không dùng Helm, ta thường làm:

```
k8s/
 |
 +-- dev/
 |    |
 |    deployment.yaml
 |
 +-- prod/
      |
      deployment.yaml
```

Hai file gần như giống nhau.

Chỉ khác vài dòng:

```yaml
replicas
image
resources
environment
```

Đây gọi là:

```
Configuration duplication
```

---

# 2.3 Helm giải quyết bằng template

Helm tách:

## Template

Phần cố định:

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:
  name: {{ .Release.Name }}

spec:
  replicas: {{ .Values.replicaCount }}
```

và:

## Values

Phần thay đổi:

```yaml
replicaCount: 3
```

Khi deploy:

```bash
helm install backend ./chart
```

Helm render thành:

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:
  name: backend

spec:
  replicas: 3
```

---

# 3. Helm hoạt động như thế nào?

Luồng hoạt động:

```text
             Helm Chart

                 |
                 |
                 v

        Template Engine

                 |
                 |
                 v

       Kubernetes YAML

                 |
                 |
                 v

          Kubernetes API Server

                 |
                 |
                 v

          Pod / Service / PVC
```

Chi tiết:

## Bước 1

Developer tạo Chart:

```
my-app/
 |
 +-- Chart.yaml
 |
 +-- values.yaml
 |
 +-- templates/
```

---

## Bước 2

User chạy:

```bash
helm install my-release ./my-app
```

---

## Bước 3

Helm đọc:

```
Chart.yaml
values.yaml
templates/*
```

---

## Bước 4

Helm render template.

Ví dụ:

Template:

```yaml
replicas: {{ .Values.replicaCount }}
```

Values:

```yaml
replicaCount: 5
```

Kết quả:

```yaml
replicas: 5
```

---

## Bước 5

Helm gửi manifest đến Kubernetes API Server.

---

# 4. Các khái niệm quan trọng trong Helm

Helm có 4 khái niệm bắt buộc phải hiểu:

```text
Chart
Release
Repository
Values
```

---

# 5. Chart là gì?

## 5.1 Định nghĩa

**Chart là một package chứa toàn bộ tài nguyên Kubernetes cần thiết để deploy một ứng dụng.**

Có thể hiểu:

```text
Chart = Application package
```

Ví dụ một chart backend:

```
backend-chart/

├── Chart.yaml

├── values.yaml

└── templates/

    ├── deployment.yaml

    ├── service.yaml

    ├── ingress.yaml

    └── configmap.yaml
```

Chart chứa:

```text
Deployment
Service
ConfigMap
Secret
Ingress
PVC
RBAC
```

---

# 5.2 Ví dụ thực tế

Không dùng Helm:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f ingress.yaml
```

Dùng Helm:

```bash
helm install payment ./payment-chart
```

Một câu lệnh thay thế hàng chục câu lệnh kubectl.

---

# 6. Release là gì?

Đây là khái niệm rất quan trọng.

## 6.1 Chart chưa phải ứng dụng đang chạy

Ví dụ:

Ta có chart:

```
mysql-chart
```

Chart chỉ là template.

Nó chưa tạo gì trong cluster.

Khi chạy:

```bash
helm install database mysql-chart
```

Helm tạo:

```
Release
```

Tên release:

```
database
```

---

Mối quan hệ:

```
Chart
 |
 |
 +---- Release 1
 |
 +---- Release 2
 |
 +---- Release 3
```

Một chart có thể tạo nhiều release.

---

Ví dụ:

Một chart nginx:

```
nginx-chart
```

Deploy:

Development:

```bash
helm install nginx-dev nginx-chart
```

Production:

```bash
helm install nginx-prod nginx-chart
```

Kết quả:

```
nginx-chart

   |
   |
   +---- nginx-dev
   |
   +---- nginx-prod
```

Hai ứng dụng dùng chung chart nhưng là hai release khác nhau.

---

# 7. Release lưu lịch sử như thế nào?

Helm quản lý revision.

Ví dụ:

Install:

```bash
helm install app ./chart
```

Revision:

```
REVISION 1
```

---

Upgrade:

```bash
helm upgrade app ./chart
```

Tạo:

```
REVISION 2
```

Upgrade tiếp:

```
REVISION 3
```

Xem lịch sử:

```bash
helm history app
```

Output:

```
REVISION   STATUS

1          deployed
2          superseded
3          deployed
```

---

Rollback:

```bash
helm rollback app 1
```

Helm quay về revision 1.

---

# 8. Repository trong Helm

## 8.1 Repository là gì?

Helm repository là nơi lưu trữ các Chart.

Tương tự:

Python:

```
PyPI
```

Node:

```
npm registry
```

Docker:

```
Docker Hub
```

Helm:

```
Chart Repository
```

---

Ví dụ:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Sau đó:

```bash
helm search repo mysql
```

Kết quả:

```
bitnami/mysql
bitnami/postgresql
bitnami/redis
```

---

# 9. Values trong Helm

## 9.1 Values là gì?

Values là file chứa configuration của chart.

File:

```
values.yaml
```

Ví dụ:

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.27"

service:
  type: ClusterIP
  port: 80
```

---

Template:

```yaml
spec:

 replicas: {{ .Values.replicaCount }}

 containers:

 - image:
     {{ .Values.image.repository }}:
     {{ .Values.image.tag }}
```

Helm lấy giá trị từ values.

---

# 9.2 Override values

Có thể override khi install:

```bash
helm install nginx ./chart \
--set replicaCount=5
```

Hoặc dùng file:

```
values-prod.yaml
```

Ví dụ:

```yaml
replicaCount: 10

resources:

 limits:

  memory: 8Gi
```

Deploy:

```bash
helm install nginx ./chart \
-f values-prod.yaml
```

---

# 10. Cấu trúc một Helm Chart

Tạo chart:

```bash
helm create my-app
```

Helm sinh:

```
my-app/

├── Chart.yaml

├── values.yaml

├── charts/

├── templates/

│
├── NOTES.txt

└── .helmignore
```

---

# 11. Chart.yaml

Đây là metadata của chart.

Ví dụ:

```yaml
apiVersion: v2

name: backend

description: Backend application

type: application

version: 1.0.0

appVersion: "2.0"
```

---

Ý nghĩa:

## apiVersion

Version của Helm chart format.

Hiện tại:

```yaml
apiVersion: v2
```

---

## name

Tên chart:

```yaml
name: backend
```

---

## version

Version của chart:

```yaml
version: 1.0.0
```

Không phải version app.

---

## appVersion

Version ứng dụng:

```yaml
appVersion: "2.0"
```

Ví dụ:

Chart:

```
backend-chart 1.0.0
```

chạy:

```
backend application 2.5
```

---

# 12. values.yaml

Đây là file cấu hình chính.

Ví dụ:

```yaml
replicaCount: 3


image:

 repository: nginx

 tag: "1.27"


service:

 type: ClusterIP

 port: 80
```

Template sẽ đọc:

```yaml
.Values.replicaCount
```

---

# 13. templates/

Đây là nơi chứa Kubernetes manifest template.

Ví dụ:

```
templates/

deployment.yaml

service.yaml

configmap.yaml

secret.yaml
```

Helm sẽ render tất cả file trong đây.

---

Ví dụ:

templates/deployment.yaml

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

 name: {{ .Release.Name }}

spec:

 replicas: {{ .Values.replicaCount }}
```

---

Nếu values:

```yaml
replicaCount: 3
```

Release:

```text
backend
```

Render:

```yaml
metadata:

 name: backend


spec:

 replicas: 3
```

---

# 14. Helm Template Engine

Helm sử dụng:

```
Go Template Language
```

để tạo YAML động.

Ví dụ:

Template:

```yaml
name: {{ .Release.Name }}
```

Input:

```
helm install api ./chart
```

Output:

```yaml
name: api
```

---

# 15. Các object quan trọng trong Helm

## 15.1 `.Values`

Lấy dữ liệu từ values.yaml.

Ví dụ:

values:

```yaml
replicaCount: 3
```

template:

```yaml
replicas: {{ .Values.replicaCount }}
```

---

## 15.2 `.Release`

Thông tin release hiện tại.

Ví dụ:

```yaml
name: {{ .Release.Name }}
```

Install:

```bash
helm install backend ./chart
```

Output:

```yaml
name: backend
```

---

## 15.3 `.Chart`

Thông tin chart.

Ví dụ:

```yaml
chart:
 {{ .Chart.Name }}
```

Output:

```
backend
```

---

## 15.4 `.Capabilities`

Kiểm tra Kubernetes version hoặc API support.

Ví dụ:

```yaml
{{ .Capabilities.KubeVersion.Version }}
```

Có thể dùng để render khác nhau giữa các cluster.

---

# 16. Ví dụ tạo Helm Chart đơn giản

Tạo chart:

```bash
helm create nginx-chart
```

Cấu trúc:

```
nginx-chart/

Chart.yaml

values.yaml

templates/

 deployment.yaml

 service.yaml
```

---

values.yaml:

```yaml
replicaCount: 2

image:

 repository: nginx

 tag: "1.27"
```

---

deployment template:

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

 name: {{ .Release.Name }}


spec:

 replicas: {{ .Values.replicaCount }}

 selector:

  matchLabels:

   app: {{ .Release.Name }}


 template:

  metadata:

   labels:

    app: {{ .Release.Name }}


  spec:

   containers:

   - name: nginx

     image:

      {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

---

Render thử:

```bash
helm template nginx ./nginx-chart
```

Helm sẽ in ra YAML Kubernetes hoàn chỉnh mà chưa deploy.

---

# 17. helm template dùng để làm gì?

Đây là command cực kỳ quan trọng khi debug.

Ví dụ:

```bash
helm template my-app ./chart
```

Giúp kiểm tra:

```text
Template render đúng chưa?
Variable có tồn tại không?
YAML có hợp lệ không?
```

Trước khi chạy:

```bash
helm install
```

Production thường luôn test:

```bash
helm template
```

hoặc:

```bash
helm lint
```

---

# 18. helm lint

Kiểm tra chart.

Command:

```bash
helm lint ./my-chart
```

Ví dụ lỗi:

```
values.yaml missing
invalid template
```

Helm sẽ báo trước khi deploy.

---

# 19. Helm Repository: Nơi lưu trữ và chia sẻ Helm Chart

Tuy nhiên, một câu hỏi đặt ra:

> Nếu muốn sử dụng một ứng dụng Kubernetes có sẵn như PostgreSQL, Redis, Kafka, Prometheus thì có cần tự viết toàn bộ YAML không?

Câu trả lời là **không**.

Helm có một hệ thống gọi là **Helm Repository**.

---

# 19.1 Helm Repository là gì?

Helm Repository là nơi lưu trữ các Helm Chart để người dùng có thể tìm kiếm và cài đặt.

Nó tương tự:

| Công nghệ  | Repository      |
| ---------- | --------------- |
| Python     | PyPI            |
| JavaScript | npm             |
| Docker     | Docker Hub      |
| Kubernetes | Helm Repository |

Ví dụ:

Thay vì tự viết:

```text
deployment.yaml
service.yaml
configmap.yaml
secret.yaml
statefulset.yaml
pvc.yaml
```

cho PostgreSQL.

Ta có thể dùng Chart có sẵn:

```bash
helm install postgres bitnami/postgresql
```

Helm sẽ tự render toàn bộ manifest Kubernetes.

---

# 19.2 Thêm Helm Repository

Ví dụ thêm Bitnami repository:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Kiểm tra repository:

```bash
helm repo list
```

Kết quả:

```text
NAME       URL
bitnami    https://charts.bitnami.com/bitnami
```

---

# 19.3 Update Helm Repository

Helm không tự biết Chart mới.

Cần update:

```bash
helm repo update
```

Ví dụ:

```text
Hang tight while we grab the latest from your chart repositories...
Update Complete.
```

Sau đó Helm có thể thấy version mới.

---

# 19.4 Tìm kiếm Chart

Tìm PostgreSQL:

```bash
helm search repo postgresql
```

Ví dụ:

```text
NAME                    CHART VERSION
bitnami/postgresql      16.4.0
```

Tìm Kafka:

```bash
helm search repo kafka
```

Tìm Redis:

```bash
helm search repo redis
```

---

# 19.5 Cài đặt Chart từ Repository

Ví dụ:

```bash
helm install my-postgres bitnami/postgresql
```

Trong câu lệnh này:

```text
my-postgres
=
Release name


bitnami/postgresql
=
Chart name
```

Helm tạo:

```text
Release:
my-postgres


Resources:

Deployment/StatefulSet
Service
Secret
PVC
ConfigMap
```

Kiểm tra:

```bash
helm list
```

Kết quả:

```text
NAME
my-postgres
```

---

# 19.6 Xem thông tin Chart

Trước khi install, nên xem Chart.

```bash
helm show chart bitnami/postgresql
```

Ví dụ:

```text
name: postgresql
version: 16.4.0
description: PostgreSQL packaged by Bitnami
```

Xem toàn bộ values:

```bash
helm show values bitnami/postgresql
```

Đây là bước rất quan trọng.

Vì mỗi Chart có hàng trăm option.

Ví dụ:

```yaml
auth:
  postgresPassword:

primary:
  persistence:
    enabled:

resources:
  requests:
```

---

# 20. Values trong Helm

Một trong những điểm mạnh nhất của Helm là **Values**.

---

# 20.1 Vấn đề khi dùng YAML truyền thống

Giả sử có môi trường:

```text
dev
staging
production
```

Nếu dùng Kubernetes YAML bình thường:

Ta có:

```text
deployment-dev.yaml

deployment-staging.yaml

deployment-prod.yaml
```

Ví dụ:

Dev:

```yaml
replicas: 1
```

Production:

```yaml
replicas: 10
```

Database:

Dev:

```yaml
storage: 10Gi
```

Production:

```yaml
storage: 500Gi
```

Ta phải copy rất nhiều file.

Dễ xảy ra:

```text
config lệch nhau
copy nhầm
khó maintain
```

---

# 20.2 Helm Values giải quyết vấn đề

Helm tách:

```text
Template
+
Configuration
```

Template:

```yaml
replicas: {{ .Values.replicaCount }}
```

Values:

```yaml
replicaCount: 3
```

Khi render:

```yaml
replicas: 3
```

---

# 20.3 File values.yaml

Ví dụ:

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.27"

service:
  type: ClusterIP
  port: 80
```

Template:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}

containers:
- image:
    {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

Sau khi render:

```yaml
replicas: 3

image:
 nginx:1.27
```

---

# 20.4 Override Values khi install

Có 3 cách.

---

## Cách 1: values.yaml

```bash
helm install web ./chart
```

Helm đọc:

```text
chart/values.yaml
```

---

## Cách 2: file values riêng

Ví dụ:

production:

```yaml
replicaCount: 10
```

Install:

```bash
helm install web ./chart \
-f production-values.yaml
```

---

## Cách 3: --set

Override trực tiếp:

```bash
helm install web ./chart \
--set replicaCount=5
```

Ví dụ:

```bash
helm install nginx bitnami/nginx \
--set service.type=LoadBalancer
```

---

# 21. Helm Upgrade

Một ứng dụng production luôn thay đổi.

Ví dụ:

Version cũ:

```text
nginx:1.27
```

Muốn update:

```text
nginx:1.28
```

Không cần:

```bash
kubectl edit deployment
```

Mà dùng:

```bash
helm upgrade
```

---

Ví dụ:

```bash
helm upgrade web ./chart \
--set image.tag=1.28
```

Helm:

```text
Render template mới

↓

So sánh với release cũ

↓

Apply thay đổi

↓

Tạo revision mới
```

---

# 21.1 Xem lịch sử Upgrade

```bash
helm history web
```

Ví dụ:

```text
REVISION
1       nginx 1.27
2       nginx 1.28
3       nginx 1.29
```

Mỗi lần upgrade tạo một revision.

---

# 22. Helm Rollback

Nếu version mới lỗi:

Ví dụ:

Deploy:

```text
nginx:1.29
```

Nhưng Pod crash:

```text
CrashLoopBackOff
```

Rollback:

```bash
helm rollback web 2
```

Helm đưa release về revision 2.

Kiểm tra:

```bash
helm history web
```

---

# 23. Helm trong môi trường Production

Trong production, Helm thường kết hợp với:

```text
Git
CI/CD
ArgoCD
Kubernetes
```

Mô hình phổ biến:

```
Developer
    |
    |
Git Repository
    |
    |
CI/CD Pipeline
    |
    |
Helm Upgrade
    |
    |
Kubernetes Cluster
```

---

Ví dụ:

Developer sửa:

```yaml
values-prod.yaml
```

Push Git:

```bash
git push
```

Pipeline:

```bash
helm upgrade app ./chart \
-f values-prod.yaml
```

Kubernetes tự cập nhật.

---

# 24. Helm và GitOps

Một mô hình hiện đại là GitOps.

Ví dụ dùng:

Argo CD

Ý tưởng:

Git là nguồn sự thật:

```
Git Repository

      |
      |
      v

ArgoCD

      |
      |
      v

Kubernetes
```

Nếu cluster khác Git:

ArgoCD phát hiện:

```text
Desired state != Actual state
```

và tự sync.

---

# 25. Helm và Secret

Một vấn đề quan trọng:

Không nên commit secret vào Git.

Ví dụ:

Sai:

values.yaml

```yaml
password: mypassword123
```

Vì Git history sẽ lưu lại.

---

Cách tốt hơn:

Dùng:

```text
Kubernetes Secret

External Secret

Vault

Sealed Secret
```

Ví dụ:

values:

```yaml
password:
  existingSecret: postgres-secret
```

Secret nằm ngoài Helm.

---

# 26. Helm Hooks

Helm có cơ chế gọi là Hook.

Hook cho phép chạy Kubernetes resource ở thời điểm đặc biệt.

Ví dụ:

```text
Before install

After install

Before upgrade

After rollback
```

---

Ví dụ:

Chạy migration database trước khi deploy app.

Luồng:

```
helm upgrade

      |
      |
      v

Database migration Job

      |
      |
      v

Deploy application
```

---

Ví dụ annotation:

```yaml
metadata:
  annotations:
    helm.sh/hook: pre-upgrade
```

Nghĩa là:

Job này chạy trước upgrade.

---

# 27. Helm Dependency

Một application thường phụ thuộc nhiều component.

Ví dụ:

Một hệ thống:

```
Application

 |
 +-- PostgreSQL

 |
 +-- Redis

 |
 +-- Kafka
```

Ta có thể khai báo dependency.

Chart.yaml:

```yaml
dependencies:

- name: postgresql
  version: 16.x
  repository:
    https://charts.bitnami.com/bitnami
```

Sau đó:

```bash
helm dependency update
```

Helm tự tải Chart phụ thuộc.

---

# 28. Helm Lint

Trước khi deploy nên kiểm tra Chart.

```bash
helm lint ./my-chart
```

Ví dụ lỗi:

```text
YAML parse error
missing required field
invalid template
```

Giúp phát hiện lỗi trước production.

---

# 29. Helm Template Debug

Một lỗi phổ biến:

Template render sai.

Ví dụ:

```yaml
replicas:
{{ .Values.replica }}
```

Nhưng values:

```yaml
replicaCount: 3
```

Sai key.

---

Dùng:

```bash
helm template app ./chart
```

Helm sẽ render YAML nhưng không deploy.

---

Debug tốt hơn:

```bash
helm install app ./chart \
--dry-run \
--debug
```

Nó cho xem:

```text
Manifest generated
Values applied
Template result
```

---

# 30. So sánh Kubernetes YAML và Helm

| Kubernetes YAML       | Helm               |
| --------------------- | ------------------ |
| Static                | Dynamic template   |
| Copy nhiều file       | Một Chart          |
| Khó multi environment | Dễ dùng values     |
| Update thủ công       | helm upgrade       |
| Rollback khó          | Revision có sẵn    |
| Không có package      | Có package manager |

---

Ví dụ:

Không dùng Helm:

```
app-dev.yaml
app-prod.yaml
service-dev.yaml
service-prod.yaml
```

Dùng Helm:

```
chart/

 templates/

 values.yaml

 values-dev.yaml

 values-prod.yaml
```

---

# 31. Helm Workflow thực tế

Một workflow chuẩn:

## Bước 1

Tạo Chart:

```bash
helm create myapp
```

## Bước 2

Sửa template:

```
templates/
```

---

## Bước 3

Test render:

```bash
helm template myapp .
```

---

## Bước 4

Lint:

```bash
helm lint .
```

---

## Bước 5

Install:

```bash
helm install myapp .
```

---

## Bước 6

Update:

```bash
helm upgrade myapp .
```

---

## Bước 7

Rollback:

```bash
helm rollback myapp 1
```

---

# 32. Lab thực hành Helm cơ bản

## Lab 1: Tạo Chart

```bash
helm create nginx-chart
```

Cấu trúc:

```
nginx-chart

Chart.yaml

values.yaml

templates/

    deployment.yaml

    service.yaml
```

---

## Lab 2: Render template

```bash
helm template nginx-chart
```

---

## Lab 3: Install

```bash
helm install nginx nginx-chart
```

Kiểm tra:

```bash
kubectl get pods
```

---

## Lab 4: Upgrade replicas

values:

```yaml
replicaCount: 5
```

Upgrade:

```bash
helm upgrade nginx nginx-chart
```

Kiểm tra:

```bash
kubectl get pods
```

---

## Lab 5: Rollback

Xem history:

```bash
helm history nginx
```

Rollback:

```bash
helm rollback nginx 1
```

---

# 33. Troubleshooting Helm

## 33.1 Release bị stuck

Kiểm tra:

```bash
helm list
```

Xem:

```text
pending-upgrade
pending-install
```

---

Xem detail:

```bash
helm status app
```

---

## 33.2 Template lỗi

Dùng:

```bash
helm template app .
```

hoặc:

```bash
helm install app . --dry-run --debug
```

---

## 33.3 Upgrade fail

Xem:

```bash
helm history app
```

Rollback:

```bash
helm rollback app revision
```

---

## 33.4 Resource đã tồn tại

Ví dụ:

```text
Service already exists
```

Nguyên nhân:

Resource được tạo ngoài Helm.

Kiểm tra:

```bash
kubectl get svc
```

Có thể cần:

```bash
helm uninstall app
```

hoặc import resource.

---

# 34. Checklist cần nắm sau bài Helm

Sau khi học Helm cần hiểu:

```
Helm là gì?

Chart là gì?

Release là gì?

Template hoạt động thế nào?

values.yaml dùng làm gì?

helm install làm gì?

helm upgrade làm gì?

helm rollback làm gì?

Repository Helm là gì?

Dependency Chart là gì?

Hook dùng khi nào?

helm template dùng để debug gì?

Helm khác kubectl apply thế nào?

Tại sao production nên dùng Helm?
```

---

# 35. Tổng kết

Helm là package manager của Kubernetes.

Nó giải quyết vấn đề:

```text
Kubernetes YAML quá nhiều
Khó quản lý version
Khó deploy nhiều môi trường
Khó rollback
```

Helm đưa ra mô hình:

```
Chart
 |
 |
Template + Values
 |
 |
Release
 |
 |
Kubernetes Resources
```

Luồng triển khai:

```
Developer

   |
   v

Helm Chart

   |
   v

helm install / upgrade

   |
   v

Kubernetes API Server

   |
   v

Pods / Services / ConfigMaps / Secrets
```

Trong môi trường production hiện nay, một stack Kubernetes thường bao gồm:

```
Docker
+
Kubernetes
+
Helm
+
CI/CD
+
GitOps
+
Monitoring
+
Logging
```

Nắm vững Helm là bước quan trọng để chuyển từ việc **tự viết YAML thủ công** sang cách vận hành Kubernetes chuyên nghiệp.
