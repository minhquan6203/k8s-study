# RBAC và ServiceAccount trong Kubernetes: Kiểm soát quyền truy cập trong Cluster

Ở các bài trước, chúng ta đã học:

* **Pod, Deployment, ReplicaSet**: cách Kubernetes quản lý workload stateless.
* **Service, DNS, Namespace**: cách các ứng dụng tìm thấy nhau trong cluster.
* **ConfigMap, Secret**: cách quản lý cấu hình và thông tin nhạy cảm.
* **Volume, PVC, StorageClass**: cách lưu trữ dữ liệu bền vững.
* **Resource Requests, Limits, Quota**: cách kiểm soát tài nguyên CPU, memory.
* **StatefulSet, Kafka, MinIO, Database**: cách chạy các hệ thống stateful.

Tuy nhiên, còn một vấn đề cực kỳ quan trọng:

> Ai được phép làm gì trong Kubernetes cluster?

Ví dụ:

* Developer có được phép xóa Pod production không?
* Team Data có được tạo PVC nhưng không được sửa Secret không?
* Một ứng dụng chạy trong Pod có được phép gọi Kubernetes API không?
* Một CI/CD pipeline có được phép deploy vào namespace production không?

Nếu Kubernetes không có cơ chế phân quyền, bất kỳ ai có quyền truy cập cluster đều có thể:

```text
kubectl delete namespace production

kubectl delete all --all

kubectl get secret -A

kubectl exec vào container production
```

Đây là rủi ro rất lớn.

Để giải quyết vấn đề này, Kubernetes sử dụng:

```text
Authentication
        |
        v
Authorization
        |
        v
RBAC
        |
        v
ServiceAccount
```

Trong đó:

```text
Authentication = Bạn là ai?

Authorization = Bạn được phép làm gì?

RBAC = Cơ chế định nghĩa quyền

ServiceAccount = Identity dành cho ứng dụng chạy trong Pod
```

Bài này sẽ đi sâu vào:

```text
1. Authentication trong Kubernetes
2. Authorization trong Kubernetes
3. RBAC là gì
4. Role và ClusterRole
5. RoleBinding và ClusterRoleBinding
6. ServiceAccount
7. Pod sử dụng ServiceAccount như thế nào
8. RBAC cho ứng dụng thực tế
9. RBAC cho CI/CD
10. Troubleshooting permission
```

---

# 1. Authentication và Authorization trong Kubernetes

Trước khi hiểu RBAC, cần hiểu Kubernetes kiểm tra request như thế nào.

Khi chạy:

```bash
kubectl get pods
```

thực chất bạn đang gửi request tới Kubernetes API Server.

Luồng:

```text
kubectl
   |
   |
   v
Kubernetes API Server
   |
   |
   +--> Authentication
   |
   +--> Authorization
   |
   +--> Admission Controller
   |
   v
Execute request
```

---

# 2. Authentication: Bạn là ai?

Authentication trả lời câu hỏi:

> Request này đến từ ai?

Ví dụ:

```text
User Minh
ServiceAccount backend-app
CI/CD Jenkins
Cloud IAM user
```

Kubernetes hỗ trợ nhiều cách authentication.

---

## 2.1 Certificate Authentication

Trong Kubernetes, user có thể sử dụng client certificate.

Ví dụ:

```text
developer.crt
developer.key
```

kubectl dùng certificate này gửi tới API Server.

API Server kiểm tra:

```text
Certificate có hợp lệ không?
Issuer có được tin tưởng không?
```

Nếu hợp lệ:

```text
User = developer
```

---

## 2.2 Token Authentication

Có thể dùng token.

Ví dụ:

```bash
kubectl config set-credentials user \
--token=<token>
```

Token được gửi trong request:

```http
Authorization: Bearer token
```

---

## 2.3 ServiceAccount Authentication

Đây là loại thường dùng nhất cho ứng dụng trong Kubernetes.

Ví dụ:

Một Pod backend muốn gọi Kubernetes API:

```text
backend-pod
      |
      |
      v
Kubernetes API Server
```

Pod cần identity.

Identity đó chính là:

```text
ServiceAccount
```

---

# 3. Authorization: Bạn được làm gì?

Sau khi Kubernetes biết:

```text
Ai đang gửi request?
```

nó cần kiểm tra:

```text
Người này có quyền thực hiện hành động đó không?
```

Ví dụ:

User:

```text
developer
```

Request:

```bash
kubectl delete pod payment-api
```

Kubernetes kiểm tra:

```text
developer có quyền delete Pod không?
```

Nếu có:

```text
ALLOW
```

Nếu không:

```text
FORBIDDEN
```

Ví dụ lỗi:

```text
Error from server (Forbidden):

pods "payment-api" is forbidden:
User "developer" cannot delete resource "pods"
```

---

# 4. RBAC là gì?

RBAC viết tắt:

```
Role Based Access Control
```

Có nghĩa:

> Phân quyền dựa trên vai trò.

Thay vì cấp quyền trực tiếp cho từng user:

```text
Minh:
  được get pod
  được create deployment
  được delete service
```

Ta tạo một role:

```text
developer-role
```

Rồi gán cho nhiều user:

```text
developer-role
        |
        +---- Minh
        |
        +---- Quan
        |
        +---- An
```

Ưu điểm:

* Quản lý dễ hơn.
* Không duplicate permission.
* Phù hợp môi trường enterprise.

---

# 5. RBAC gồm những thành phần nào?

RBAC Kubernetes có 4 object chính:

```text
Role

ClusterRole

RoleBinding

ClusterRoleBinding
```

Quan hệ:

```text
             Permission

                 |
                 v

        Role / ClusterRole

                 |
                 v

       RoleBinding / ClusterRoleBinding

                 |
                 v

       User / Group / ServiceAccount
```

---

# 6. Role trong Kubernetes

## 6.1 Role là gì?

Role định nghĩa permission trong **một namespace cụ thể**.

Ví dụ:

Namespace:

```text
development
```

Role:

```text
developer-role
```

Cho phép:

```text
get pods
create deployments
update services
```

Nhưng chỉ trong namespace:

```text
development
```

---

Ví dụ:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:
  name: developer-role
  namespace: development

rules:
- apiGroups:
  - ""

  resources:
  - pods

  verbs:
  - get
  - list
  - watch
```

Ý nghĩa:

Trong namespace:

```text
development
```

Role này cho phép:

```text
get Pod
list Pod
watch Pod
```

---

# 7. Resource và Verb trong RBAC

RBAC permission gồm:

```yaml
resources:
verbs:
```

---

## 7.1 Resources

Resource là object Kubernetes.

Ví dụ:

```text
pods

services

deployments

configmaps

secrets

persistentvolumeclaims

jobs
```

---

Ví dụ:

```yaml
resources:
- pods
- services
```

nghĩa là:

```text
Có quyền trên Pod và Service
```

---

## 7.2 Verbs

Verb là hành động.

Các verb phổ biến:

| Verb   | Ý nghĩa           |
| ------ | ----------------- |
| get    | xem một object    |
| list   | xem danh sách     |
| watch  | theo dõi thay đổi |
| create | tạo               |
| update | sửa               |
| patch  | cập nhật một phần |
| delete | xóa               |

---

Ví dụ:

```yaml
verbs:
- get
- list
- create
```

nghĩa là:

```text
Có thể xem và tạo
Không được xóa
```

---

# 8. RoleBinding

Role chỉ định nghĩa quyền.

Nhưng chưa gán quyền cho ai.

Ví dụ:

Role:

```text
developer-role
```

cho phép:

```text
get pods
```

Nhưng Kubernetes chưa biết:

```text
Ai được dùng role này?
```

RoleBinding giải quyết vấn đề đó.

---

Ví dụ:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:
  name: developer-binding
  namespace: development

subjects:

- kind: User
  name: minh

roleRef:

  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io
```

Ý nghĩa:

User:

```text
minh
```

được sử dụng:

```text
developer-role
```

trong namespace:

```text
development
```

---

# 9. ClusterRole

Role chỉ có phạm vi namespace.

Nhưng có những quyền cần áp dụng toàn cluster.

Ví dụ:

Admin cần xem:

```text
Pod ở tất cả namespace
```

Không thể tạo 100 Role cho 100 namespace.

Ta dùng:

```text
ClusterRole
```

---

Ví dụ:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

metadata:
  name: pod-reader

rules:

- apiGroups:
  - ""

  resources:
  - pods

  verbs:
  - get
  - list
```

Role này có hiệu lực toàn cluster.

---

# 10. ClusterRoleBinding

ClusterRole cần được gán bằng:

```text
ClusterRoleBinding
```

Ví dụ:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding

metadata:
  name: pod-reader-binding

subjects:

- kind: User
  name: minh


roleRef:

  kind: ClusterRole
  name: pod-reader
  apiGroup:
    rbac.authorization.k8s.io
```

Kết quả:

User:

```text
minh
```

có quyền:

```text
get/list Pod
```

trên:

```text
toàn cluster
```

---

# 11. Role vs ClusterRole

So sánh:

|          | Role          | ClusterRole             |
| -------- | ------------- | ----------------------- |
| Scope    | Namespace     | Toàn cluster            |
| Dùng cho | App/team nhỏ  | Admin/system            |
| Binding  | RoleBinding   | ClusterRoleBinding      |
| Ví dụ    | Dev namespace | Monitoring toàn cluster |

---

# 12. RoleBinding vs ClusterRoleBinding

|                 | RoleBinding | ClusterRoleBinding |
| --------------- | ----------- | ------------------ |
| Phạm vi         | Namespace   | Cluster            |
| Gán Role        | Có          | Không              |
| Gán ClusterRole | Có          | Có                 |
| Dùng cho        | Developer   | Admin              |

---

Một điểm hay:

RoleBinding có thể bind ClusterRole.

Ví dụ:

ClusterRole:

```text
view
```

Nhưng RoleBinding:

```text
namespace dev
```

Kết quả:

User chỉ có quyền view trong namespace dev.

---

# 13. ServiceAccount là gì?

User thường là con người.

Ví dụ:

```text
developer
admin
data-team
```

Nhưng ứng dụng cũng cần identity.

Ví dụ:

Một Pod:

```text
spark-job
```

muốn đọc Secret:

```text
database-password
```

hoặc gọi Kubernetes API:

```text
GET /api/v1/pods
```

Pod cần identity.

Đó là:

```text
ServiceAccount
```

---

# 14. Tạo ServiceAccount

Ví dụ:

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: backend-sa
  namespace: production
```

Apply:

```bash
kubectl apply -f sa.yaml
```

Kiểm tra:

```bash
kubectl get serviceaccount -n production
```

---

# 15. Gán ServiceAccount cho Pod

Mặc định Pod dùng:

```text
default ServiceAccount
```

Ví dụ:

```yaml
spec:

  serviceAccountName: backend-sa

  containers:

  - name: backend
    image: backend:v1
```

Pod này chạy dưới identity:

```text
backend-sa
```

---

# 16. Ví dụ thực tế: Backend đọc Kubernetes API

Giả sử backend cần xem Pod.

Flow:

```text
Backend Pod

      |
      |
      v

ServiceAccount

      |
      |
      v

RBAC Permission

      |
      |
      v

Kubernetes API Server
```

---

## Bước 1: ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:
  name: backend-sa
  namespace: app
```

---

## Bước 2: Role

Cho phép đọc Pod:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role

metadata:

  name: pod-reader

  namespace: app


rules:

- apiGroups:
  - ""

  resources:
  - pods

  verbs:

  - get
  - list
```

---

## Bước 3: RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding

metadata:

  name: backend-binding

  namespace: app


subjects:

- kind: ServiceAccount

  name: backend-sa

  namespace: app


roleRef:

  kind: Role

  name: pod-reader

  apiGroup:
    rbac.authorization.k8s.io
```

---

## Bước 4: Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:

  name: backend


spec:

  template:

    spec:

      serviceAccountName:
        backend-sa

      containers:

      - name: backend

        image:
          backend:v1
```

---

Bây giờ backend có quyền:

```text
GET pods
LIST pods
```

nhưng không thể:

```text
DELETE pods
```

vì RBAC không cấp.

---

# 17. Default ServiceAccount

Mỗi namespace tự động có:

```text
default ServiceAccount
```

Kiểm tra:

```bash
kubectl get sa
```

Ví dụ:

```text
NAME
default
```

Nếu Pod không khai báo:

```yaml
serviceAccountName:
```

nó dùng:

```text
default
```

---

Trong production:

Không nên để app quan trọng dùng default SA.

Nên:

```text
Tạo ServiceAccount riêng
+
Cấp đúng quyền cần thiết
```

Đây gọi là:

```
Least Privilege Principle
```

---

# 18. Least Privilege Principle

Nguyên tắc:

> Cấp đúng quyền cần thiết, không cấp dư.

Sai:

```yaml
verbs:
- "*"

resources:
- "*"
```

Nghĩa:

```text
Toàn quyền cluster
```

Rất nguy hiểm.

---

Đúng:

Backend chỉ cần đọc ConfigMap:

```yaml
resources:
- configmaps

verbs:
- get
```

Không cấp:

```text
delete
create
```

---

# 19. Kubernetes mặc định RBAC

Cluster Kubernetes thường có các ClusterRole mặc định:

Xem:

```bash
kubectl get clusterrole
```

Một số role:

```text
cluster-admin

admin

edit

view
```

---

## cluster-admin

Toàn quyền:

```text
*
*
```

Rất nguy hiểm.

---

## view

Chỉ đọc:

```text
get
list
watch
```

---

## edit

Có thể:

```text
create
update
delete
```

Nhưng không quản lý RBAC.

---

## admin

Quản lý namespace.

---

# 20. Kiểm tra quyền với kubectl auth can-i

Lệnh cực kỳ quan trọng:

```bash
kubectl auth can-i
```

---

Ví dụ:

User có được delete Pod không?

```bash
kubectl auth can-i delete pods
```

Kết quả:

```text
yes
```

hoặc:

```text
no
```

---

Kiểm tra trong namespace:

```bash
kubectl auth can-i create deployments \
-n production
```

---

Kiểm tra ServiceAccount:

```bash
kubectl auth can-i get pods \
--as=system:serviceaccount:app:backend-sa
```

---

# 21. RBAC trong CI/CD

Một pipeline:

```text
GitHub Actions

      |
      v

kubectl apply

      |
      v

Kubernetes
```

CI/CD cần quyền.

Không nên dùng:

```text
cluster-admin
```

---

Tạo ServiceAccount:

```text
github-actions
```

Cấp quyền:

```text
deploy app namespace production
```

Ví dụ:

```yaml
resources:

- deployments
- services

verbs:

- create
- update
- patch
```

Pipeline chỉ có thể deploy.

Không thể:

```text
delete namespace
read secret
```

---

# 22. RBAC và Secret

Một lỗi bảo mật phổ biến:

Cấp:

```yaml
resources:
- secrets

verbs:
- get
```

cho quá nhiều user.

Vì Secret chứa:

```text
Database password
API key
Token
Certificate
```

Ví dụ:

```bash
kubectl get secret db-password -o yaml
```

có thể lấy credential.

---

Best practice:

```text
Chỉ ServiceAccount cần dùng Secret mới được đọc
```

---

# 23. Troubleshooting RBAC

## 23.1 Forbidden error

Ví dụ:

```text
Error from server (Forbidden)

User cannot get resource pods
```

Kiểm tra:

```bash
kubectl auth can-i get pods
```

---

## 23.2 Kiểm tra Role

```bash
kubectl get role -n namespace
```

---

## 23.3 Kiểm tra RoleBinding

```bash
kubectl get rolebinding -n namespace
```

---

## 23.4 Xem chi tiết

```bash
kubectl describe rolebinding name
```

---

## 23.5 Kiểm tra ServiceAccount

```bash
kubectl get sa
```

---

# 24. Bài lab tự làm

## Lab 1: Tạo ServiceAccount

Tạo:

```text
backend-sa
```

trong namespace:

```text
demo
```

---

## Lab 2: Tạo Role đọc Pod

Permission:

```text
get/list/watch pods
```

---

## Lab 3: Bind Role cho ServiceAccount

Tạo:

```text
RoleBinding
```

---

## Lab 4: Deploy Pod sử dụng ServiceAccount

Kiểm tra:

```bash
kubectl describe pod
```

xem:

```text
Service Account
```

---

## Lab 5: Test quyền

Kiểm tra:

```bash
kubectl auth can-i get pods \
--as=system:serviceaccount:demo:backend-sa
```

Kết quả:

```text
yes
```

---

Test:

```bash
kubectl auth can-i delete pods \
--as=system:serviceaccount:demo:backend-sa
```

Kết quả:

```text
no
```

---

# 25. Checklist cần nắm sau bài này

Sau bài RBAC và ServiceAccount, cần hiểu:

```text
Authentication là gì?

Authorization là gì?

RBAC dùng để làm gì?

Role khác ClusterRole thế nào?

RoleBinding khác ClusterRoleBinding thế nào?

Resource và Verb trong RBAC là gì?

ServiceAccount dùng cho ai?

Pod sử dụng ServiceAccount như thế nào?

Vì sao không nên dùng default ServiceAccount?

Least privilege là gì?

kubectl auth can-i dùng để làm gì?

CI/CD nên dùng ServiceAccount như thế nào?

Làm sao debug lỗi Forbidden?
```

---

# 26. Tóm tắt ngắn gọn

RBAC kiểm soát:

```text
Ai được làm gì trong Kubernetes
```

Công thức nhớ:

```text
Subject
(User / Group / ServiceAccount)

        +

Role
(Permission)

        +

Binding
(Gán quyền)

        =

Access
```

Ví dụ:

```text
backend-sa

+

Role:
get pods

+

RoleBinding

=

Backend Pod được đọc Pod trong namespace
```

Phân biệt:

```text
Role
= quyền trong namespace

ClusterRole
= quyền toàn cluster

RoleBinding
= gán quyền trong namespace

ClusterRoleBinding
= gán quyền toàn cluster
```

ServiceAccount:

```text
Identity cho application chạy trong Pod
```

Best practice production:

```text
Không dùng default ServiceAccount

Không cấp cluster-admin cho app

Cấp quyền tối thiểu

Tách ServiceAccount theo workload

Audit RBAC thường xuyên
```

RBAC là một trong những phần quan trọng nhất khi đưa Kubernetes vào production, bởi vì Kubernetes không chỉ là hệ thống chạy container, mà còn là một nền tảng quản lý hạ tầng với rất nhiều quyền truy cập. Hiểu RBAC giúp đảm bảo cluster vừa linh hoạt vừa an toàn.
