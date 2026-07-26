# 14. CI/CD Deploy Kubernetes: Tự động hóa Build, Test và Deploy ứng dụng lên Kubernetes

Ở các bài trước, chúng ta đã tìm hiểu Kubernetes quản lý workload như thế nào:

* Deployment quản lý ứng dụng stateless.
* StatefulSet quản lý workload cần storage và identity ổn định.
* Service giúp các ứng dụng giao tiếp với nhau.
* ConfigMap và Secret quản lý cấu hình.
* Helm giúp đóng gói và triển khai ứng dụng.
* Monitoring giúp quan sát hệ thống.

Tuy nhiên, trong môi trường production, một câu hỏi quan trọng xuất hiện:

> Khi developer thay đổi code, làm sao đưa version mới của ứng dụng lên Kubernetes một cách tự động, an toàn và có thể rollback?

Ví dụ:

Developer sửa code:

```text
Developer

    |

    |

git push

    |

    |

Build Docker Image

    |

    |

Push Image Registry

    |

    |

Deploy Kubernetes

    |

    |

Application Updated
```

Nếu làm thủ công:

```bash
docker build
docker push
kubectl apply
kubectl rollout status
```

thì sẽ có nhiều vấn đề:

```text
- Dễ sai command
- Deploy chậm
- Khó audit
- Khó rollback
- Không kiểm soát được version
```

Giải pháp là:

```text
CI/CD
```

---

# 1. CI/CD là gì?

CI/CD là viết tắt của:

```text
CI  = Continuous Integration

CD  = Continuous Delivery / Continuous Deployment
```

Đây là quy trình tự động hóa việc:

```text
Code

↓

Build

↓

Test

↓

Package

↓

Deploy

↓

Monitor
```

---

# 2. Continuous Integration (CI)

## 2.1 CI là gì?

Continuous Integration nghĩa là:

> Mỗi khi developer đưa code mới vào repository, hệ thống tự động build và kiểm tra code.

Ví dụ:

Developer:

```bash
git push origin main
```

Pipeline chạy:

```text
1. Checkout code

2. Install dependency

3. Run unit test

4. Build Docker image

5. Scan security

6. Push image
```

---

Mục tiêu của CI:

```text
Phát hiện lỗi càng sớm càng tốt
```

Ví dụ:

Developer commit:

```python
def divide(a,b):
    return a/b
```

Test phát hiện:

```text
division by zero error
```

Pipeline fail.

Code chưa được deploy.

---

# 3. Continuous Delivery và Continuous Deployment

Hai khái niệm này thường bị nhầm.

---

# 3.1 Continuous Delivery

Continuous Delivery nghĩa là:

```text
Code luôn ở trạng thái có thể deploy
```

Pipeline:

```text
Git Push

    |

Build

    |

Test

    |

Create Image

    |

Ready for Deploy
```

Nhưng bước deploy production cần người approve.

Ví dụ:

```text
Deploy to Dev       Auto

Deploy to Staging   Auto

Deploy Production   Manual Approval
```

---

# 3.2 Continuous Deployment

Continuous Deployment tự động hoàn toàn:

```text
Git Push

    |

CI pass

    |

Auto Deploy Production
```

Không cần người approve.

---

So sánh:

|                   | Continuous Delivery | Continuous Deployment |
| ----------------- | ------------------- | --------------------- |
| Build             | Auto                | Auto                  |
| Test              | Auto                | Auto                  |
| Deploy staging    | Auto                | Auto                  |
| Deploy production | Manual              | Auto                  |

---

# 4. CI/CD trong Kubernetes

Kubernetes không tự build code.

Kubernetes chỉ làm nhiệm vụ:

```text
Run container

Manage lifecycle

Scale

Networking

Storage
```

CI/CD pipeline chịu trách nhiệm:

```text
Build application

Create Docker image

Push image

Update Kubernetes manifest
```

---

Kiến trúc tổng quan:

```
                 Developer


                     |

                     |

                 Git Repository


                     |

                     |

                    CI


        +------------+------------+

        |                         |

      Build                    Test


        |

        |

 Docker Image


        |

        |

 Container Registry


        |

        |

       CD


        |

        |

 Kubernetes Cluster


        |

        |

 Application Pods
```

---

# 5. Các thành phần trong Kubernetes CI/CD

Một pipeline Kubernetes thường có:

```
Git Repository

        |

CI Server

        |

Docker Registry

        |

Kubernetes Manifest

        |

Kubernetes API Server

        |

Cluster
```

---

# 6. Git Repository

Nơi lưu:

```
Source code

Dockerfile

Kubernetes YAML

Helm Chart

Pipeline config
```

Ví dụ:

```
my-app/


├── src/

├── Dockerfile

├── requirements.txt

├── helm/

│
└── .github/

    └── workflows/
```

---

Các Git platform phổ biến:

```
GitHub

GitLab

Bitbucket
```

---

# 7. CI Server

CI Server là hệ thống chạy pipeline.

Ví dụ:

```
GitHub Actions

GitLab CI/CD

Jenkins

Argo Workflows

Tekton
```

---

Nó thực hiện:

```
Checkout code

Build

Test

Docker build

Push image
```

---

# 8. Docker Image trong CI/CD

Kubernetes không chạy source code.

Kubernetes chạy:

```
Container Image
```

Ví dụ:

Application:

```
Python FastAPI
```

Dockerfile:

```dockerfile
FROM python:3.11

WORKDIR /app

COPY requirements.txt .

RUN pip install -r requirements.txt

COPY .

CMD ["python","main.py"]
```

---

CI build:

```bash
docker build \
-t my-app:v1 .
```

Kết quả:

```
my-app:v1
```

---

Push registry:

```bash
docker push my-registry/my-app:v1
```

---

# 9. Container Registry

Registry lưu Docker image.

Ví dụ:

```
Docker Hub

GitHub Container Registry

GitLab Container Registry

AWS ECR

Google Artifact Registry

Harbor
```

---

Luồng:

```
Developer

    |

Docker Build

    |

Image

    |

Registry

    |

Kubernetes Pull Image

```

---

Ví dụ Kubernetes Deployment:

```yaml
spec:
  containers:

  - name: api

    image:
      registry.example.com/api:v1
```

Kubernetes sẽ pull image:

```
registry.example.com/api:v1
```

---

# 10. Kubernetes Deployment trong CI/CD

Ví dụ Deployment:

```yaml
apiVersion: apps/v1

kind: Deployment


metadata:

  name: api


spec:

  replicas: 3


  selector:

    matchLabels:

      app: api


  template:

    metadata:

      labels:

        app: api


    spec:

      containers:

      - name: api

        image:
          my-api:v1
```

---

Khi release version mới:

Image:

```
my-api:v2
```

CI/CD cập nhật:

```yaml
image:
  my-api:v2
```

---

Kubernetes thực hiện:

```
Deployment

      |

Create new ReplicaSet

      |

Create new Pods

      |

Terminate old Pods
```

Đây chính là:

```
Rolling Update
```

---

# 11. Rolling Update trong CI/CD

Kubernetes mặc định dùng:

```
RollingUpdate
```

Ví dụ:

Version cũ:

```
api:v1


Pod 1
Pod 2
Pod 3
```

Deploy:

```
api:v2
```

Kubernetes:

```
Create:

api:v2 Pod

        |

Check Ready

        |

Remove api:v1 Pod

        |

Repeat
```

---

Ưu điểm:

```
Không downtime

Deploy an toàn

Có thể rollback
```

---

# 12. Rollback trong CI/CD

Ví dụ:

Deploy:

```
api:v2
```

Nhưng lỗi:

```
Application crash

500 error

Memory leak
```

Rollback:

```bash
kubectl rollout undo deployment/api
```

Kubernetes quay lại:

```
api:v1
```

---

Trong production:

CI/CD phải luôn giữ:

```
Previous image version

Previous Helm revision

Deployment history
```

---

# 13. CI Pipeline Example

Ví dụ GitHub Actions:

File:

```
.github/workflows/ci.yaml
```

---

```yaml
name: CI


on:

  push:

    branches:

    - main



jobs:


  build:


    runs-on: ubuntu-latest


    steps:


    - name:
        Checkout

      uses:
        actions/checkout@v4



    - name:
        Build Docker Image

      run:

        docker build \
        -t my-api:${{github.sha}} .



    - name:
        Push Image

      run:

        docker push my-api:${{github.sha}}
```

---

Luồng:

```
git push

 |

GitHub Actions

 |

docker build

 |

docker push

```

---

# 14. CD Pipeline Example

Sau khi có image:

```
my-api:abc123
```

CD deploy:

```bash
kubectl set image deployment/api \
api=my-api:abc123
```

---

Hoặc:

```bash
helm upgrade api ./chart \
--set image.tag=abc123
```

---

# 15. Kubernetes Deployment Strategies

Không phải lúc nào cũng dùng Rolling Update.

Có nhiều strategy.

---

# 15.1 Recreate Strategy

Xóa toàn bộ version cũ:

```
Stop old version

      |

Deploy new version
```

Nhược điểm:

```
Downtime
```

Dùng cho:

```
Batch job

Internal service
```

---

# 15.2 Rolling Update

Mặc định.

```
Old

 |

New
```

Ưu điểm:

```
Zero downtime
```

---

# 15.3 Blue Green Deployment

Có hai môi trường:

```
Blue:

version v1


Green:

version v2
```

Ban đầu:

```
Traffic

 |

Blue
```

Test Green:

```
Green OK

```

Switch:

```
Traffic

 |

Green
```

---

Ưu điểm:

```
Rollback cực nhanh
```

Nhược điểm:

```
Tốn gấp đôi resource
```

---

# 15.4 Canary Deployment

Deploy một phần traffic.

Ví dụ:

```
90%

v1


10%

v2
```

Theo dõi:

```
Error rate

Latency

CPU
```

Nếu tốt:

```
100% v2
```

---

Canary thường dùng với:

```
Istio

Argo Rollouts

NGINX Ingress
```

---

# 16. GitOps là gì?

Đây là mô hình CI/CD hiện đại trong Kubernetes.

Nguyên tắc:

```
Git là source of truth
```

---

Thay vì:

```
CI

 |

kubectl apply
```

GitOps:

```
Git Repository

        |

        |

GitOps Controller

        |

        |

Kubernetes
```

---

Controller tự động sync:

```
Git state

     |

Cluster state
```

---

Công cụ phổ biến:

```
ArgoCD

FluxCD
```

---

# 17. ArgoCD trong Kubernetes

ArgoCD là GitOps tool phổ biến nhất.

Kiến trúc:

```
Git Repository


      |

      |

   ArgoCD


      |

      |

Kubernetes API


      |

      |

Cluster
```

---

Ví dụ Git chứa:

```
helm/

deployment.yaml

service.yaml
```

ArgoCD thấy:

```
Git:

replicas=5


Cluster:

replicas=3
```

Nó tự sync:

```
replicas=5
```

---

# 18. Helm + CI/CD

Trong production thường không deploy YAML trực tiếp.

Dùng:

```
Helm Chart
```

Ví dụ:

```
my-app-chart/


Chart.yaml

values.yaml

templates/

    deployment.yaml

    service.yaml

```

---

CI/CD:

```
Build image

      |

Push image

      |

Update values.yaml

      |

helm upgrade

      |

Kubernetes
```

---

Ví dụ:

values.yaml:

```yaml
image:

 repository:
   my-api


 tag:
   v2
```

CD update:

```bash
helm upgrade api ./chart
```

---

# 19. GitLab CI/CD thực tế với Kubernetes

Trong thực tế doanh nghiệp, GitLab CI/CD là một trong những hệ thống được sử dụng rất nhiều để build và deploy Kubernetes.

GitLab cung cấp:

```text
Git Repository

+

CI/CD Pipeline

+

Container Registry

+

Deployment Automation
```

---

Kiến trúc tổng quát:

```text
Developer

    |

    |

git push

    |

    |

GitLab Repository

    |

    |

GitLab Runner

    |

    |

Build Docker Image

    |

    |

Push GitLab Container Registry

    |

    |

Deploy Kubernetes

    |

    |

Production Cluster
```

---

# 20. GitLab Runner là gì?

GitLab Runner là agent thực thi pipeline.

GitLab chỉ lưu pipeline definition.

Runner mới là thành phần chạy:

```text
docker build

docker push

kubectl apply

helm upgrade

test command
```

---

Ví dụ:

Developer push code:

```bash
git push origin main
```

GitLab nhận event.

Sau đó gửi job cho Runner:

```text
GitLab

 |

 |

Runner

 |

 |

Execute Script
```

---

Runner có thể chạy dưới nhiều dạng:

```text
Shell Runner

Docker Runner

Kubernetes Runner
```

---

# 20.1 Kubernetes GitLab Runner

Trong môi trường Kubernetes, thường deploy GitLab Runner ngay trong cluster.

Mô hình:

```text
Kubernetes Cluster


Namespace gitlab-runner


        |

        |

GitLab Runner Pod


        |

        |

Create temporary CI Job Pods

```

---

Mỗi pipeline job có thể tạo một Pod riêng:

```text
Pipeline Job

      |

      |

Temporary Pod

      |

      |

Build/Test/Deploy

      |

      |

Pod deleted
```

---

Ưu điểm:

```text
Không cần server CI riêng

Scale runner tự động

Isolation tốt
```

---

# 21. Ví dụ GitLab CI Pipeline

File:

```text
.gitlab-ci.yml
```

Ví dụ:

```yaml
stages:

  - test

  - build

  - deploy



test:

  stage: test

  script:

    - pytest



build:

  stage: build

  script:

    - docker build -t my-api:$CI_COMMIT_SHA .

    - docker push my-api:$CI_COMMIT_SHA



deploy:

  stage: deploy

  script:

    - kubectl set image deployment/api \
      api=my-api:$CI_COMMIT_SHA
```

---

Pipeline:

```text
Stage 1

Test


      ↓


Stage 2

Build Image


      ↓


Stage 3

Deploy Kubernetes

```

---

# 22. Kubernetes Authentication trong CI/CD

Một vấn đề quan trọng:

CI/CD làm sao có quyền deploy Kubernetes?

Ví dụ:

GitLab Runner chạy:

```bash
kubectl apply -f deployment.yaml
```

Nhưng Kubernetes API Server cần xác thực.

---

Có hai cách phổ biến:

```text
kubeconfig

hoặc

ServiceAccount + RBAC
```

---

# 22.1 Dùng kubeconfig

Tạo kubeconfig:

```yaml
apiVersion: v1

clusters:

- cluster:

    server:
      https://kubernetes-api


contexts:

- context:

    cluster:
      production

    user:
      gitlab


users:

- name:
    gitlab

  user:

    token:
      xxx
```

---

Trong GitLab CI:

Variable:

```text
KUBE_CONFIG
```

Load:

```bash
echo $KUBE_CONFIG > ~/.kube/config
```

Sau đó:

```bash
kubectl get pods
```

---

Nhược điểm:

```text
Khó quản lý

Token quyền lớn

Khó rotate
```

---

# 22.2 ServiceAccount + RBAC (khuyến nghị)

Tạo ServiceAccount:

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: gitlab-deployer

  namespace: cicd
```

---

Role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1

kind: Role


metadata:

  name: deploy-role

  namespace: production


rules:


- apiGroups:

    - apps

  resources:

    - deployments

  verbs:

    - get

    - update

    - patch
```

---

RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1

kind: RoleBinding


metadata:

  name: deploy-binding

  namespace: production


subjects:


- kind:
    ServiceAccount

  name:
    gitlab-deployer


roleRef:


  kind:
    Role

  name:
    deploy-role

```

---

Bây giờ CI chỉ có quyền:

```text
Update Deployment

Không thể:

Delete Namespace

Read Secret

Modify RBAC
```

---

Đây gọi là:

```text
Principle of Least Privilege
```

---

# 23. Docker Registry Authentication trong Kubernetes

Kubernetes cần pull image private.

Ví dụ:

```yaml
image:

private-registry.com/api:v1
```

Nhưng registry yêu cầu login.

---

Giải pháp:

```text
imagePullSecret
```

---

Tạo secret:

```bash
kubectl create secret docker-registry registry-secret \
--docker-server=registry.example.com \
--docker-username=user \
--docker-password=password
```

---

Deployment:

```yaml
spec:

  imagePullSecrets:


  - name:
      registry-secret


  containers:


  - name:
      api


    image:
      registry.example.com/api:v1
```

---

Luồng:

```text
Pod start

 |

 |

Kubernetes đọc imagePullSecret

 |

 |

Login Registry

 |

 |

Pull Image

```

---

# 24. Quản lý Environment trong CI/CD

Production thường không chỉ có một môi trường.

Thường có:

```text
Development

Staging

Production
```

---

Ví dụ:

```text
Developer

    |

    |

dev


    |

    |

staging


    |

    |

production

```

---

Mỗi môi trường có:

```text
Namespace riêng

Config riêng

Secret riêng

Database riêng

Resource limit riêng
```

---

Ví dụ:

```text
dev namespace:

api-dev

mysql-dev


staging namespace:

api-staging

mysql-staging


prod namespace:

api-prod

mysql-prod

```

---

# 25. CI/CD với nhiều Namespace

Ví dụ:

```text
namespace dev

    |

    |

helm install api ./chart

```

Deploy:

```bash
helm upgrade api ./chart \
-n dev
```

---

Production:

```bash
helm upgrade api ./chart \
-n production
```

---

Cùng một Helm Chart:

```text
chart

 |

 +---- dev values

 |

 +---- staging values

 |

 +---- prod values

```

---

Ví dụ:

values-dev.yaml:

```yaml
replicaCount: 1

resources:

  cpu: 100m
```

---

values-prod.yaml:

```yaml
replicaCount: 5

resources:

  cpu: 1000m
```

---

Deploy:

Dev:

```bash
helm upgrade api ./chart \
-f values-dev.yaml
```

Production:

```bash
helm upgrade api ./chart \
-f values-prod.yaml
```

---

# 26. Secrets trong CI/CD

Một lỗi rất nguy hiểm:

Không được commit secret vào Git.

Sai:

```yaml
database:

 password:
   mypassword123
```

---

Đúng:

```text
Git

 |

 |

Reference Secret

 |

 |

Kubernetes Secret
```

---

Ví dụ:

Kubernetes Secret:

```yaml
apiVersion: v1

kind: Secret

metadata:

 name: database-secret


stringData:

 username:
   admin


 password:
   password123
```

---

Deployment:

```yaml
env:


- name:

    DB_PASSWORD


  valueFrom:


    secretKeyRef:


      name:
        database-secret


      key:
        password
```

---

Trong CI/CD:

Secret nên nằm trong:

```text
GitLab CI Variables

GitHub Secrets

Vault

AWS Secrets Manager

Hashicorp Vault
```

---

# 27. GitOps Workflow với ArgoCD

CI/CD truyền thống:

```text
Developer

 |

Git Push

 |

CI Build

 |

kubectl apply

 |

Cluster
```

---

GitOps:

```text
Developer

 |

Git Push


 |

CI Build Image


 |

Update Manifest Repo


 |

ArgoCD Detect Change


 |

Sync Cluster

```

---

Điểm khác biệt:

CI không trực tiếp vào Kubernetes.

Thay vào đó:

```text
Git Repository
=
Desired State
```

---

# 28. ArgoCD Application

Ví dụ:

ArgoCD Application:

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application


metadata:

 name: api


spec:


 source:


   repoURL:
      https://github.com/company/k8s-config


   path:
      production/api


 destination:


   server:
      https://kubernetes.default.svc


   namespace:
      production
```

---

ArgoCD liên tục kiểm tra:

```text
Git State

vs

Cluster State
```

---

Nếu khác:

Ví dụ:

Git:

```yaml
replicas:5
```

Cluster:

```yaml
replicas:3
```

ArgoCD:

```text
OutOfSync

Sync required
```

---

# 29. CI/CD kết hợp Helm + ArgoCD

Đây là kiến trúc production phổ biến.

Luồng:

```text
Developer


 |

 |

Application Repo


 |

 |

CI Pipeline


 |

 |

Build Docker Image


 |

 |

Push Registry


 |

 |

Update Helm Values Repo


 |

 |

ArgoCD


 |

 |

Kubernetes

```

---

Ví dụ:

App repo:

```text
backend-code
```

Config repo:

```text
k8s-config

 |

 helm-chart

 |

 values-prod.yaml
```

---

CI update:

```yaml
image:

 tag:
   v2.0.1
```

---

ArgoCD tự deploy.

---

# 30. Monitoring sau khi Deploy

CI/CD không kết thúc sau deploy.

Cần kiểm tra:

```text
Application Health

Pod Status

Error Rate

Latency

Resource Usage
```

---

Ví dụ:

Deploy xong:

```bash
kubectl get pods
```

OK:

```text
api-5d8d9 Running
```

Nhưng cần kiểm tra:

```bash
kubectl logs api-5d8d9
```

---

Production thường kết hợp:

```text
Prometheus

Grafana

AlertManager

Loki

ELK Stack
```

---

# 31. Deployment Failure trong CI/CD

Một pipeline có thể fail ở nhiều bước.

---

## Build fail

Ví dụ:

```text
Docker build error
```

Kiểm tra:

```text
Dockerfile

Dependency

Build log
```

---

## Push fail

Ví dụ:

```text
unauthorized
```

Nguyên nhân:

```text
Registry credential sai
```

---

## Deploy fail

Ví dụ:

```text
ImagePullBackOff
```

Nguyên nhân:

```text
Image không tồn tại

Registry auth lỗi
```

---

## Kubernetes rollout fail

Ví dụ:

```text
ProgressDeadlineExceeded
```

Nguyên nhân:

```text
Pod không Ready

Health check fail

Resource thiếu
```

---

# 32. Rollback trong CI/CD

Rollback thủ công:

```bash
kubectl rollout undo deployment/api
```

---

Rollback bằng Helm:

```bash
helm rollback api 3
```

---

Rollback bằng GitOps:

Revert commit:

```bash
git revert commit-id
```

ArgoCD:

```text
Detect change

↓

Sync previous version

```

---

# 33. Best Practices CI/CD Kubernetes

## 1. Không deploy bằng tay

Tránh:

```bash
kubectl apply
```

trên production.

Nên:

```text
Pipeline

hoặc

GitOps
```

---

## 2. Tag image bằng version rõ ràng

Không dùng:

```yaml
image:
 latest
```

Nên:

```yaml
image:
 api:v1.2.3
```

---

## 3. Immutable Deployment

Không sửa image cũ:

Sai:

```text
api:v1

replace content
```

Đúng:

```text
api:v1

api:v2

api:v3
```

---

## 4. Tự động rollback

Có thể kết hợp:

```text
Health Check

+

Deployment Strategy

+

Monitoring
```

---

## 5. Phân quyền CI/CD

Không cho CI:

```text
cluster-admin
```

Nên:

```text
namespace permission

specific resource permission
```

---

# 34. Checklist bài 14

Sau bài này cần hiểu:

```text
CI là gì?

CD là gì?

Continuous Delivery khác Continuous Deployment thế nào?

CI/CD pipeline Kubernetes gồm những bước nào?

Docker image được build ở đâu?

Registry dùng để làm gì?

GitLab Runner là gì?

CI authenticate Kubernetes bằng cách nào?

ServiceAccount trong CI/CD dùng thế nào?

imagePullSecret là gì?

Helm được dùng trong CI/CD ra sao?

Dev/Staging/Production quản lý thế nào?

GitOps là gì?

ArgoCD hoạt động thế nào?

Rolling Update là gì?

Blue Green Deployment là gì?

Canary Deployment là gì?

Rollback Kubernetes thực hiện thế nào?
```

---

# 35. Tổng kết

CI/CD biến Kubernetes từ một hệ thống deploy thủ công thành một nền tảng triển khai tự động.

Luồng hoàn chỉnh trong production:

```text
Developer

   |

   |

Git Push

   |

   |

CI Pipeline

   |

   |

Run Test

   |

   |

Build Docker Image

   |

   |

Push Registry

   |

   |

Update Helm Manifest

   |

   |

ArgoCD Sync

   |

   |

Kubernetes Deployment

   |

   |

Rolling Update

   |

   |

Monitoring

   |

   |

Rollback nếu lỗi
```

Khi kết hợp:

```text
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
```

ta có một nền tảng DevOps hoàn chỉnh để vận hành ứng dụng production ở quy mô lớn.
