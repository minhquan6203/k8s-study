# Pod, ReplicaSet và Deployment trong Kubernetes: Hiểu đúng nền tảng workload stateless

Khi bắt đầu học Kubernetes, ba khái niệm rất dễ gặp là **Pod**, **ReplicaSet** và **Deployment**. Đây là nhóm kiến thức nền tảng nhất để hiểu cách Kubernetes chạy một ứng dụng container hóa.

Nếu Docker giúp ta đóng gói và chạy một container, thì Kubernetes giúp ta quản lý rất nhiều container trên nhiều node khác nhau. Tuy nhiên, Kubernetes không quản lý container trực tiếp theo kiểu đơn giản là “chạy container A”. Thay vào đó, Kubernetes quản lý workload thông qua các object như Pod, ReplicaSet và Deployment.

Trong bài này, ta sẽ đi từ lý thuyết cốt lõi đến cách các object này phối hợp với nhau khi scale, update version và rollback ứng dụng.

---

## 1. Vì sao cần hiểu Pod, ReplicaSet và Deployment?

Trong thực tế, khi deploy một ứng dụng lên Kubernetes, ta thường không chỉ muốn chạy một container duy nhất. Ta thường cần nhiều yêu cầu hơn:

Ứng dụng phải luôn chạy.

Nếu một instance chết, hệ thống phải tự tạo instance mới.

Có thể chạy nhiều bản sao để chịu tải tốt hơn.

Có thể update version mới mà không làm downtime toàn bộ hệ thống.

Nếu version mới lỗi, có thể rollback về version cũ.

Đó chính là lý do Kubernetes không chỉ có một object duy nhất. Nó chia trách nhiệm ra thành nhiều lớp:

```text
Deployment
   |
   v
ReplicaSet
   |
   v
Pod
   |
   v
Container
```

Mỗi lớp có một nhiệm vụ riêng:

```text
Pod = đơn vị chạy container
ReplicaSet = đảm bảo số lượng Pod luôn đúng
Deployment = quản lý version, rollout và rollback
```

Khi chạy app thông thường, đặc biệt là workload stateless như API, web server, worker không lưu trạng thái cục bộ quan trọng, ta gần như luôn dùng **Deployment** thay vì tạo Pod hoặc ReplicaSet trực tiếp.

---

## 2. Stateless workload là gì?

Trước khi đi sâu vào Pod, ReplicaSet và Deployment, cần hiểu khái niệm **stateless workload**.

Một workload được xem là stateless khi mỗi instance của nó không cần giữ dữ liệu riêng quan trọng bên trong container hoặc local disk. Nếu Pod bị xóa rồi tạo lại, ứng dụng vẫn có thể hoạt động bình thường.

Ví dụ stateless workload:

```text
REST API
Web backend
Frontend server
Nginx
FastAPI service
Spring Boot service
Node.js service
Background worker đọc từ queue
```

Các ứng dụng này thường lưu dữ liệu ở bên ngoài:

```text
Database
Redis
Kafka
S3/MinIO
Object Storage
External API
Persistent Volume
```

Vì vậy, một Pod API chết đi và Pod mới được tạo lại thì không sao, miễn là Pod mới vẫn kết nối được tới database hoặc service bên ngoài.

Ngược lại, workload stateful là những hệ thống cần định danh ổn định, storage ổn định hoặc thứ tự khởi động rõ ràng, ví dụ:

```text
Kafka
PostgreSQL
MySQL
Elasticsearch
ScyllaDB
Zookeeper
MinIO distributed mode
```

Với các workload stateful, ta thường dùng **StatefulSet** thay vì Deployment. Nhưng trong bài này, ta tập trung vào workload stateless với Pod, ReplicaSet và Deployment.

---

# 3. Pod là gì?

## 3.1 Pod là đơn vị deploy nhỏ nhất trong Kubernetes

Trong Kubernetes, **Pod** là đơn vị nhỏ nhất có thể được Kubernetes schedule và chạy trên node.

Một Pod có thể chứa một hoặc nhiều container. Trường hợp phổ biến nhất là một Pod chứa một container chính.

Ví dụ:

```text
Pod web-abc123
   |
   v
Container nginx
```

Hoặc trong một số trường hợp nâng cao, một Pod có thể chứa nhiều container phối hợp chặt với nhau:

```text
Pod app-with-sidecar
   |
   +-- Container main-app
   +-- Container log-sidecar
   +-- Container proxy-sidecar
```

Các container trong cùng một Pod sẽ chia sẻ một số tài nguyên quan trọng, đặc biệt là network namespace.

---

## 3.2 Các container trong cùng Pod dùng chung network

Một điểm quan trọng là các container trong cùng Pod dùng chung network namespace.

Điều này có nghĩa là:

```text
Các container trong cùng Pod dùng chung IP
Các container có thể gọi nhau qua localhost
Các container không được bind trùng port
```

Ví dụ một Pod có hai container:

```text
Pod my-pod
   |
   +-- app container chạy port 8080
   +-- sidecar container chạy port 9000
```

Container sidecar có thể gọi app container bằng:

```text
http://localhost:8080
```

Vì chúng cùng nằm trong một Pod và dùng chung network namespace.

Tuy nhiên, trong thực tế khi mới học Kubernetes, ta nên bắt đầu với mô hình đơn giản nhất:

```text
1 Pod = 1 container app
```

Sau khi hiểu chắc Pod, Deployment, Service rồi mới học sidecar pattern.

---

## 3.3 Pod có IP riêng nhưng không bền vững

Mỗi Pod khi được tạo sẽ có một IP riêng trong cluster.

Ví dụ:

```text
web-abc123   10.244.1.10
web-def456   10.244.2.15
web-xyz789   10.244.3.20
```

Tuy nhiên, Pod IP không bền vững.

Nếu Pod chết, bị xóa, bị evict khỏi node hoặc Deployment tạo Pod mới, Pod mới có thể có IP khác.

Ví dụ:

```text
Pod cũ:
web-abc123   10.244.1.10

Pod bị xóa.

Pod mới:
web-jkl999   10.244.2.30
```

Vì vậy, trong Kubernetes, ta không nên để client gọi trực tiếp vào Pod IP.

Nếu một service khác gọi trực tiếp:

```text
http://10.244.1.10:80
```

thì khi Pod bị tạo lại và IP đổi, request sẽ fail.

Để giải quyết vấn đề này, Kubernetes dùng object **Service** để cung cấp địa chỉ ổn định phía trước các Pod. Service sẽ được học riêng, nhưng ở đây cần nhớ một ý:

```text
Pod IP có thể đổi.
Service IP/DNS ổn định hơn.
```

---

## 3.4 Pod không phải object lý tưởng để chạy app lâu dài

Ta có thể tạo Pod trực tiếp bằng manifest YAML. Tuy nhiên, trong thực tế, ta hiếm khi tạo Pod trực tiếp cho ứng dụng production.

Lý do là Pod đơn lẻ không tự đảm bảo số lượng bản sao mong muốn theo cách đầy đủ như Deployment.

Ví dụ ta tạo một Pod nginx trực tiếp. Nếu Pod đó bị xóa, tùy cách tạo và restart policy, nó có thể không được tạo lại theo đúng mong muốn quản lý lâu dài. Pod trực tiếp cũng không hỗ trợ rollout version, rollback, revision history.

Vì vậy:

```text
Pod trực tiếp: phù hợp cho debug, test nhanh, chạy tạm thời.
Deployment: phù hợp để chạy app thật, lâu dài.
```

Ví dụ dùng Pod trực tiếp khi cần debug:

```bash
kubectl run debug --image=busybox -it --rm -- sh
```

Còn khi deploy API, web app, worker, ta nên dùng Deployment.

---

# 4. ReplicaSet là gì?

## 4.1 ReplicaSet đảm bảo số lượng Pod mong muốn

**ReplicaSet** là object có nhiệm vụ đảm bảo luôn có đúng số lượng Pod mong muốn.

Ví dụ ta muốn chạy 3 Pod của app web:

```text
Desired replicas = 3
```

ReplicaSet sẽ quan sát trạng thái thực tế trong cluster.

Nếu hiện tại chỉ có 2 Pod đang chạy:

```text
Current replicas = 2
```

ReplicaSet sẽ tạo thêm 1 Pod.

Nếu hiện tại có 4 Pod:

```text
Current replicas = 4
```

ReplicaSet sẽ xóa bớt 1 Pod.

Cơ chế này gọi là **reconciliation loop**.

Nói đơn giản, Kubernetes controller liên tục so sánh:

```text
desired state = trạng thái mong muốn
actual state = trạng thái thực tế
```

Nếu hai trạng thái khác nhau, Kubernetes sẽ hành động để kéo actual state về gần desired state.

---

## 4.2 Ví dụ về cơ chế tự bù Pod

Giả sử ReplicaSet được khai báo:

```text
replicas = 3
```

Ban đầu có 3 Pod:

```text
web-1
web-2
web-3
```

Nếu `web-2` chết:

```text
web-1
web-3
```

ReplicaSet phát hiện chỉ còn 2 Pod, trong khi desired state là 3. Nó sẽ tạo Pod mới:

```text
web-1
web-3
web-4
```

Điểm quan trọng là ReplicaSet không nhất thiết “hồi sinh” đúng Pod cũ. Nó tạo một Pod mới để đảm bảo số lượng.

Vì vậy, Pod nên được thiết kế theo hướng stateless. Không nên phụ thuộc vào việc Pod cụ thể nào đó tồn tại mãi mãi.

---

## 4.3 ReplicaSet chọn Pod bằng label selector

ReplicaSet không quản lý Pod bằng tên Pod cụ thể. Nó quản lý Pod thông qua **label selector**.

Ví dụ ReplicaSet có selector:

```yaml
selector:
  matchLabels:
    app: web
```

Điều đó có nghĩa là ReplicaSet sẽ quản lý các Pod có label:

```yaml
labels:
  app: web
```

Đây là một điểm rất quan trọng.

Kubernetes dùng label rất nhiều để liên kết các object với nhau. ReplicaSet dùng selector để biết Pod nào thuộc về nó.

Nếu selector và label không khớp, ReplicaSet sẽ không quản lý được Pod đúng cách.

Ví dụ sai:

```yaml
selector:
  matchLabels:
    app: web
```

Nhưng template Pod lại ghi:

```yaml
labels:
  app: nginx
```

Lúc này ReplicaSet tìm Pod có label `app=web`, nhưng Pod được tạo ra lại có label `app=nginx`. Đây là lỗi cấu hình nghiêm trọng.

Vì vậy cần nhớ:

```text
spec.selector.matchLabels phải khớp với template.metadata.labels.
```

---

## 4.4 Có nên tạo ReplicaSet trực tiếp không?

Thông thường là không.

Ta nên để Deployment tạo và quản lý ReplicaSet.

Lý do là ReplicaSet chỉ đảm bảo số lượng Pod, nhưng không quản lý rollout, rollback và lịch sử version tốt như Deployment.

Trong thực tế:

```text
Không nên tạo ReplicaSet trực tiếp cho app thông thường.
Nên tạo Deployment.
Deployment sẽ tự tạo ReplicaSet phía sau.
```

Khi ta chạy:

```bash
kubectl get deploy,rs,pods
```

Ta sẽ thấy Deployment, ReplicaSet và Pod cùng tồn tại.

Ví dụ:

```text
deployment.apps/web
replicaset.apps/web-7c9f8d6b8
pod/web-7c9f8d6b8-a1b2c
pod/web-7c9f8d6b8-d3e4f
pod/web-7c9f8d6b8-g5h6i
```

Ở đây:

```text
Deployment web quản lý ReplicaSet web-7c9f8d6b8.
ReplicaSet web-7c9f8d6b8 quản lý các Pod phía dưới.
```

---

# 5. Deployment là gì?

## 5.1 Deployment là object chuẩn để chạy app stateless

**Deployment** là object cấp cao hơn ReplicaSet. Nó dùng để quản lý ứng dụng stateless theo cách an toàn và có version.

Deployment chịu trách nhiệm:

```text
Tạo ReplicaSet
Đảm bảo số lượng replicas
Rolling update khi đổi image hoặc template
Rollback về revision cũ
Lưu revision history
Theo dõi trạng thái rollout
```

Với ứng dụng thông thường như web API, frontend, backend, worker, ta gần như luôn dùng Deployment.

---

## 5.2 Deployment quản lý version thông qua ReplicaSet

Khi ta tạo Deployment lần đầu với image:

```text
nginx:1.27
```

Kubernetes tạo một ReplicaSet tương ứng với template đó.

Ví dụ:

```text
Deployment web
   |
   v
ReplicaSet web-aaa111, image nginx:1.27
   |
   +-- Pod web-aaa111-1
   +-- Pod web-aaa111-2
   +-- Pod web-aaa111-3
```

Khi ta update image sang:

```text
nginx:1.28
```

Deployment không sửa trực tiếp Pod cũ. Thay vào đó, nó tạo một ReplicaSet mới cho version mới:

```text
Deployment web
   |
   +-- ReplicaSet web-aaa111, image nginx:1.27
   |
   +-- ReplicaSet web-bbb222, image nginx:1.28
```

Sau đó Deployment giảm dần số Pod ở ReplicaSet cũ và tăng dần số Pod ở ReplicaSet mới.

Đây chính là cơ chế rolling update.

---

## 5.3 Deployment giúp rollout an toàn hơn

Nếu không có Deployment, khi update version, ta có thể phải tự làm thủ công:

```text
Stop app cũ
Start app mới
Check lỗi
Nếu lỗi thì quay lại
```

Cách này dễ gây downtime.

Deployment hỗ trợ rolling update để thay đổi dần dần.

Ví dụ app đang có 3 Pod version cũ:

```text
v1-pod-1
v1-pod-2
v1-pod-3
```

Khi update sang version mới, Kubernetes có thể làm từng bước:

```text
Bước 1:
v1-pod-1
v1-pod-2
v1-pod-3
v2-pod-1

Bước 2:
v1-pod-2
v1-pod-3
v2-pod-1
v2-pod-2

Bước 3:
v1-pod-3
v2-pod-1
v2-pod-2
v2-pod-3

Bước 4:
v2-pod-1
v2-pod-2
v2-pod-3
```

Tùy cấu hình `maxSurge` và `maxUnavailable`, Kubernetes sẽ quyết định được tạo thêm bao nhiêu Pod mới và được phép thiếu bao nhiêu Pod trong quá trình update.

---

# 6. Phân tích manifest Deployment mẫu

Dưới đây là manifest Deployment mẫu:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: default
  labels:
    app: web
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "300m"
              memory: "256Mi"
```

Ta sẽ phân tích từng phần.

---

## 6.1 apiVersion và kind

```yaml
apiVersion: apps/v1
kind: Deployment
```

`apiVersion` cho biết object này thuộc API group/version nào của Kubernetes.

`kind` cho biết loại object cần tạo là gì.

Ở đây:

```text
apps/v1 = API version cho Deployment
Deployment = loại object dùng để quản lý workload stateless
```

---

## 6.2 metadata

```yaml
metadata:
  name: web
  namespace: default
  labels:
    app: web
```

`metadata.name` là tên của Deployment.

`namespace` cho biết Deployment được tạo trong namespace nào.

`labels` là nhãn gắn vào Deployment. Label giúp phân loại, tìm kiếm, chọn lọc object.

Ví dụ có thể xem Deployment theo label:

```bash
kubectl get deploy -l app=web
```

---

## 6.3 replicas

```yaml
spec:
  replicas: 3
```

`replicas: 3` nghĩa là ta muốn luôn có 3 Pod chạy cho app này.

Nếu một Pod chết, ReplicaSet tạo Pod mới.

Nếu ta scale lên 5:

```bash
kubectl scale deploy/web --replicas=5
```

Kubernetes sẽ tạo thêm Pod cho đủ 5.

Nếu ta scale xuống 2:

```bash
kubectl scale deploy/web --replicas=2
```

Kubernetes sẽ xóa bớt Pod để còn 2.

Điểm quan trọng:

```text
replicas thể hiện desired state.
Kubernetes sẽ cố gắng làm actual state khớp với desired state.
```

---

## 6.4 revisionHistoryLimit

```yaml
revisionHistoryLimit: 5
```

`revisionHistoryLimit` quy định Deployment giữ lại tối đa bao nhiêu ReplicaSet cũ để phục vụ rollback.

Ví dụ nếu để:

```yaml
revisionHistoryLimit: 5
```

Kubernetes có thể giữ lại 5 revision gần nhất.

Khi update nhiều lần, các ReplicaSet quá cũ có thể bị xóa.

Nếu muốn rollback được nhiều version trước đó, có thể tăng giá trị này. Nhưng giữ quá nhiều revision cũng làm cluster có thêm nhiều object lịch sử không cần thiết.

---

## 6.5 strategy: RollingUpdate

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

Deployment có nhiều chiến lược update, nhưng phổ biến nhất là `RollingUpdate`.

Rolling update nghĩa là Kubernetes thay Pod cũ bằng Pod mới từ từ, không dừng toàn bộ app cùng lúc.

### maxSurge là gì?

```yaml
maxSurge: 1
```

`maxSurge` là số Pod có thể vượt quá số replicas mong muốn trong lúc update.

Ví dụ `replicas: 3`, `maxSurge: 1`.

Trong lúc rollout, Kubernetes được phép chạy tối đa:

```text
3 + 1 = 4 Pod
```

Tức là có thể tạo thêm 1 Pod mới trước khi xóa Pod cũ.

### maxUnavailable là gì?

```yaml
maxUnavailable: 1
```

`maxUnavailable` là số Pod được phép không sẵn sàng trong lúc update.

Ví dụ `replicas: 3`, `maxUnavailable: 1`.

Trong lúc update, Kubernetes chấp nhận tối thiểu còn:

```text
3 - 1 = 2 Pod available
```

Điều này giúp quá trình update linh hoạt hơn.

Nếu set `maxUnavailable: 0`, Kubernetes sẽ cố đảm bảo không làm giảm số Pod available dưới số replicas mong muốn. Nhưng khi đó rollout có thể cần tài nguyên dư để tạo Pod mới trước.

---

## 6.6 selector và template labels

```yaml
selector:
  matchLabels:
    app: web
```

Selector là cách Deployment/ReplicaSet tìm các Pod thuộc về nó.

Bên dưới có template:

```yaml
template:
  metadata:
    labels:
      app: web
```

Hai phần này phải khớp nhau.

Deployment nói:

```text
Tôi quản lý các Pod có label app=web.
```

Pod template nói:

```text
Các Pod được tạo ra sẽ có label app=web.
```

Nếu hai phần này không khớp, Deployment sẽ không hoạt động đúng.

Đây là lỗi rất hay gặp khi mới học Kubernetes.

---

## 6.7 template

```yaml
template:
  metadata:
    labels:
      app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

`template` là khuôn mẫu để tạo Pod.

Deployment không trực tiếp ghi từng Pod cụ thể. Nó khai báo Pod template. ReplicaSet dùng template đó để tạo Pod.

Nếu template thay đổi, ví dụ đổi image từ `nginx:1.27` sang `nginx:1.28`, Deployment sẽ hiểu rằng đây là một version mới và tạo ReplicaSet mới.

Các thay đổi trong Pod template thường trigger rollout, ví dụ:

```text
Đổi image
Đổi command
Đổi env
Đổi resource requests/limits
Đổi label trong template
Đổi volume mount
```

---

## 6.8 container image

```yaml
image: nginx:1.27
```

Image là container image được dùng để tạo container.

Trong production, nên tránh dùng tag không rõ ràng như:

```text
latest
```

Vì `latest` không cho biết chính xác version nào đang chạy. Khi rollback hoặc debug sẽ khó kiểm soát.

Nên dùng tag cụ thể:

```text
nginx:1.27
my-api:1.0.3
my-api:2026-07-05-abc123
```

Với CI/CD, nhiều team dùng Git commit SHA làm image tag để trace được version code.

---

## 6.9 containerPort

```yaml
ports:
  - containerPort: 80
```

`containerPort` mô tả port mà container lắng nghe.

Điểm cần hiểu: `containerPort` chủ yếu mang tính khai báo metadata trong Pod spec. Nó không tự động expose app ra ngoài cluster.

Muốn service khác gọi app ổn định, cần tạo Kubernetes Service.

Ví dụ Deployment chạy nginx port 80, nhưng nếu chưa có Service, app khác không nên gọi trực tiếp vào Pod IP.

---

## 6.10 resources: requests và limits

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "300m"
    memory: "256Mi"
```

Đây là phần rất quan trọng trong Kubernetes.

### requests

`requests` là lượng tài nguyên tối thiểu mà container yêu cầu.

Scheduler dùng `requests` để quyết định Pod nên chạy trên node nào.

Ví dụ:

```yaml
requests:
  cpu: "100m"
  memory: "128Mi"
```

Nghĩa là Pod cần ít nhất 0.1 CPU core và 128 MiB RAM.

Trong Kubernetes:

```text
1000m CPU = 1 CPU core
100m CPU = 0.1 CPU core
500m CPU = 0.5 CPU core
```

Nếu node không còn đủ tài nguyên theo requests, Pod sẽ không được schedule.

### limits

`limits` là mức tối đa container được phép dùng.

Ví dụ:

```yaml
limits:
  cpu: "300m"
  memory: "256Mi"
```

Container có thể bị giới hạn CPU ở mức 0.3 core. Nếu dùng memory vượt quá 256Mi, container có thể bị kill với lỗi OOMKilled.

Cần nhớ:

```text
requests ảnh hưởng scheduling.
limits ảnh hưởng runtime limit.
```

Nếu không set requests/limits, cluster có thể khó quản lý tài nguyên. Pod có thể chiếm quá nhiều tài nguyên hoặc bị schedule không tối ưu.

---

# 7. Quan hệ giữa Deployment, ReplicaSet và Pod

Khi apply manifest Deployment:

```bash
kubectl apply -f web-deployment.yaml
```

Kubernetes tạo ra chuỗi object như sau:

```text
Deployment web
   |
   v
ReplicaSet web-xxxx
   |
   +-- Pod web-xxxx-abcde
   +-- Pod web-xxxx-fghij
   +-- Pod web-xxxx-klmno
```

Ta có thể xem bằng lệnh:

```bash
kubectl get deploy,rs,pods -l app=web -o wide
```

Kết quả sẽ cho thấy:

```text
Deployment có desired replicas.
ReplicaSet có số replicas đang quản lý.
Pod là instance thực tế đang chạy container.
```

Cách nhớ đơn giản:

```text
Deployment nghĩ về version và rollout.
ReplicaSet nghĩ về số lượng.
Pod nghĩ về việc chạy container.
```

---

# 8. Scale Deployment hoạt động như thế nào?

Khi chạy:

```bash
kubectl scale deploy/web --replicas=5
```

Ta đang thay đổi desired state của Deployment từ 3 lên 5.

Deployment cập nhật ReplicaSet hiện tại để replicas tăng lên 5.

ReplicaSet thấy hiện tại mới có 3 Pod, trong khi desired là 5, nên tạo thêm 2 Pod.

Luồng xử lý:

```text
User chạy kubectl scale
   |
   v
API Server cập nhật Deployment spec.replicas = 5
   |
   v
Deployment controller cập nhật ReplicaSet
   |
   v
ReplicaSet controller tạo thêm Pod
   |
   v
Scheduler chọn node cho Pod mới
   |
   v
Kubelet trên node pull image và chạy container
```

Khi scale xuống:

```bash
kubectl scale deploy/web --replicas=2
```

ReplicaSet sẽ xóa bớt Pod để còn đúng 2.

Điểm hay của Kubernetes là ta không cần tự chọn xóa Pod nào hoặc tạo Pod trên node nào. Kubernetes tự làm dựa trên desired state.

---

# 9. Rolling update hoạt động như thế nào?

Khi chạy:

```bash
kubectl set image deploy/web nginx=nginx:1.28
```

Ta đã thay đổi image trong Pod template.

Kubernetes nhận ra template thay đổi, nên tạo một ReplicaSet mới.

Ví dụ ban đầu:

```text
ReplicaSet old: nginx:1.27, replicas = 3
```

Sau khi update:

```text
ReplicaSet new: nginx:1.28, replicas = 0
```

Deployment sẽ bắt đầu rolling update:

```text
Tăng replicas của new ReplicaSet
Giảm replicas của old ReplicaSet
Lặp lại đến khi new ReplicaSet đạt đủ replicas
```

Với cấu hình:

```yaml
replicas: 3
maxSurge: 1
maxUnavailable: 1
```

Quá trình có thể diễn ra như sau:

```text
Ban đầu:
old = 3
new = 0
total = 3

Bước 1:
old = 3
new = 1
total = 4

Bước 2:
old = 2
new = 1
total = 3

Bước 3:
old = 2
new = 2
total = 4

Bước 4:
old = 1
new = 2
total = 3

Bước 5:
old = 1
new = 3
total = 4

Bước 6:
old = 0
new = 3
total = 3
```

Nếu các Pod mới Ready thành công, rollout hoàn tất.

Có thể theo dõi bằng:

```bash
kubectl rollout status deploy/web
```

---

# 10. Rollout fail là gì?

Rollout fail xảy ra khi version mới không thể chạy ổn định.

Ví dụ update image sang tag không tồn tại:

```bash
kubectl set image deploy/web nginx=nginx:notfound
```

Kubernetes sẽ cố tạo Pod mới, nhưng node không pull được image.

Pod có thể rơi vào trạng thái:

```text
ImagePullBackOff
ErrImagePull
```

Rollout không hoàn tất vì Pod mới không Ready.

Khi xem rollout status, có thể thấy rollout bị kẹt.

```bash
kubectl rollout status deploy/web
```

Nếu vượt quá thời gian deadline, Deployment có thể báo:

```text
ProgressDeadlineExceeded
```

Điều này thường xảy ra khi:

```text
Image không tồn tại
Image pull secret sai
App crash khi start
Readiness probe fail
Thiếu config/env
Không đủ resource
Pod không schedule được
```

---

# 11. Rollback hoạt động như thế nào?

Deployment lưu revision history thông qua các ReplicaSet cũ.

Khi rollout version mới lỗi, có thể rollback:

```bash
kubectl rollout undo deploy/web
```

Kubernetes sẽ quay lại revision trước đó.

Ví dụ:

```text
Revision 1: nginx:1.27
Revision 2: nginx:notfound
```

Nếu revision 2 lỗi, rollback sẽ đưa Deployment về revision 1.

Có thể xem lịch sử rollout:

```bash
kubectl rollout history deploy/web
```

Có thể rollback về revision cụ thể:

```bash
kubectl rollout undo deploy/web --to-revision=1
```

Điểm quan trọng: rollback không phải là “khôi phục Pod cũ đã chết”. Nó cập nhật Deployment template về cấu hình của revision cũ, sau đó Kubernetes rollout lại theo cơ chế Deployment.

---

# 12. Các trạng thái thường gặp của Pod

Khi học Deployment, cần quen với các trạng thái Pod.

## Pending

Pod đã được tạo nhưng chưa chạy.

Nguyên nhân thường gặp:

```text
Không đủ CPU/RAM trên node
Không có node phù hợp
PVC chưa bind
Taints/tolerations không khớp
Image đang pull
```

Kiểm tra bằng:

```bash
kubectl describe pod <pod-name>
```

## Running

Pod đã được schedule lên node và container đang chạy.

Nhưng Running không có nghĩa là app đã sẵn sàng nhận traffic. Cần xem thêm readiness.

## CrashLoopBackOff

Container start rồi crash liên tục.

Nguyên nhân thường gặp:

```text
App lỗi khi khởi động
Sai command
Thiếu env
Không connect được dependency
Config sai
```

Kiểm tra log:

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

## ImagePullBackOff

Kubernetes không pull được image.

Nguyên nhân thường gặp:

```text
Image tag sai
Image không tồn tại
Registry private nhưng thiếu imagePullSecret
Node không truy cập được registry
```

## OOMKilled

Container dùng quá giới hạn memory và bị kill.

Nguyên nhân:

```text
Memory limit quá thấp
App bị memory leak
Batch size quá lớn
Load tăng đột biến
```

Kiểm tra:

```bash
kubectl describe pod <pod-name>
```

---

# 13. Lệnh thực hành cơ bản

## 13.1 Apply Deployment

```bash
kubectl apply -f web-deployment.yaml
```

Lệnh này gửi manifest lên Kubernetes API Server. Kubernetes sẽ tạo hoặc cập nhật Deployment theo cấu hình trong file.

## 13.2 Xem Deployment, ReplicaSet và Pod

```bash
kubectl get deploy,rs,pods -l app=web -o wide
```

Lệnh này giúp thấy đầy đủ ba lớp:

```text
Deployment
ReplicaSet
Pod
```

`-l app=web` dùng để lọc object có label `app=web`.

`-o wide` hiển thị thêm thông tin như node, IP, image.

## 13.3 Scale Deployment

```bash
kubectl scale deploy/web --replicas=5
```

Tăng số Pod mong muốn lên 5.

Xem Pod:

```bash
kubectl get pods -l app=web
```

Scale xuống:

```bash
kubectl scale deploy/web --replicas=2
```

## 13.4 Update image

```bash
kubectl set image deploy/web nginx=nginx:1.28
```

Lệnh này đổi image của container tên `nginx` trong Deployment `web`.

Tên `nginx` ở đây là tên container trong manifest:

```yaml
containers:
  - name: nginx
```

Nếu container tên khác, lệnh phải đổi tương ứng.

## 13.5 Theo dõi rollout

```bash
kubectl rollout status deploy/web
```

Lệnh này cho biết rollout đã hoàn tất chưa.

## 13.6 Xem lịch sử rollout

```bash
kubectl rollout history deploy/web
```

Xem các revision đã từng deploy.

## 13.7 Rollback

```bash
kubectl rollout undo deploy/web
```

Rollback về revision trước.

Rollback về revision cụ thể:

```bash
kubectl rollout undo deploy/web --to-revision=1
```

---

# 14. Bài lab đề xuất

## Bước 1: Tạo Deployment web với 3 replicas

Tạo file `web-deployment.yaml` với manifest Deployment.

Apply:

```bash
kubectl apply -f web-deployment.yaml
```

Kiểm tra:

```bash
kubectl get deploy,rs,pods -l app=web -o wide
```

Quan sát:

```text
Có 1 Deployment
Có 1 ReplicaSet
Có 3 Pod
```

## Bước 2: Scale lên 5 replicas

```bash
kubectl scale deploy/web --replicas=5
```

Kiểm tra:

```bash
kubectl get pods -l app=web
```

Kỳ vọng:

```text
Số Pod tăng từ 3 lên 5
ReplicaSet hiện tại quản lý 5 Pod
```

## Bước 3: Scale xuống 2 replicas

```bash
kubectl scale deploy/web --replicas=2
```

Kỳ vọng:

```text
Kubernetes xóa bớt Pod
Còn 2 Pod running
```

## Bước 4: Update image sang version mới

```bash
kubectl set image deploy/web nginx=nginx:1.28
kubectl rollout status deploy/web
```

Quan sát:

```bash
kubectl get rs -l app=web
```

Kỳ vọng:

```text
Có ReplicaSet mới được tạo
ReplicaSet cũ giảm Pod
ReplicaSet mới tăng Pod
```

## Bước 5: Update image sang tag lỗi

```bash
kubectl set image deploy/web nginx=nginx:notfound
kubectl rollout status deploy/web
```

Quan sát Pod:

```bash
kubectl get pods -l app=web
```

Có thể thấy Pod mới bị:

```text
ImagePullBackOff
```

Kiểm tra chi tiết:

```bash
kubectl describe pod <pod-name>
```

## Bước 6: Rollback

```bash
kubectl rollout undo deploy/web
kubectl rollout status deploy/web
```

Kiểm tra lại:

```bash
kubectl get deploy,rs,pods -l app=web -o wide
```

Kỳ vọng:

```text
Deployment quay về image cũ hoạt động được
Pod lỗi bị thay thế
Rollout hoàn tất
```

---

# 15. Troubleshooting nhanh

## 15.1 Deployment rollout bị ProgressDeadlineExceeded

Lỗi này nghĩa là Deployment không đạt được tiến độ rollout trong thời gian cho phép.

Nguyên nhân thường gặp:

```text
Image sai
Container crash
Readiness probe fail
Thiếu Secret hoặc ConfigMap
Không đủ resource
Không pull được image
```

Kiểm tra:

```bash
kubectl describe deploy web
kubectl get pods -l app=web
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

## 15.2 Pod không được tạo đủ số lượng

Ví dụ Deployment khai báo 5 replicas nhưng chỉ có 3 Pod Running.

Cần kiểm tra:

```bash
kubectl get deploy web
kubectl describe deploy web
kubectl get rs -l app=web
kubectl get pods -l app=web
kubectl get events --sort-by=.lastTimestamp
```

Nguyên nhân có thể là:

```text
Namespace bị quota giới hạn CPU/RAM
Node không đủ tài nguyên
Pod Pending vì thiếu PVC
Taints/tolerations không phù hợp
Node selector không match node nào
```

## 15.3 Selector sai

Nếu `spec.selector.matchLabels` không khớp với `template.metadata.labels`, Deployment có thể không tạo hoặc không quản lý Pod đúng.

Cần đảm bảo:

```yaml
selector:
  matchLabels:
    app: web
```

phải khớp với:

```yaml
template:
  metadata:
    labels:
      app: web
```

Đây là lỗi cơ bản nhưng rất quan trọng.

## 15.4 ImagePullBackOff

Kiểm tra:

```bash
kubectl describe pod <pod-name>
```

Tìm phần Events ở cuối output.

Nguyên nhân thường là:

```text
Sai tên image
Sai tag
Registry không truy cập được
Thiếu imagePullSecret
```

## 15.5 CrashLoopBackOff

Kiểm tra log:

```bash
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

Nguyên nhân thường là app tự crash sau khi start.

Ví dụ:

```text
Thiếu biến môi trường
Không connect được database
Sai command entrypoint
Config file không tồn tại
Port bind lỗi
```

---

# 16. Những hiểu nhầm phổ biến

## 16.1 “Deployment chạy container”

Nói chính xác hơn:

```text
Deployment không trực tiếp chạy container.
Deployment tạo ReplicaSet.
ReplicaSet tạo Pod.
Pod chạy container.
```

Cách nói “Deployment chạy app” vẫn dùng được trong giao tiếp hằng ngày, nhưng khi debug cần hiểu đúng các lớp bên dưới.

## 16.2 “ReplicaSet là thứ mình nên viết YAML trực tiếp”

Thông thường không nên.

Với app stateless thông thường:

```text
Viết Deployment.
Để Deployment quản lý ReplicaSet.
```

ReplicaSet chủ yếu là object trung gian để đảm bảo số lượng Pod và hỗ trợ rollout history.

## 16.3 “Pod IP dùng để gọi service”

Không nên thiết kế app gọi trực tiếp Pod IP.

Pod IP không ổn định. Khi Pod bị recreate, IP đổi.

Nên gọi qua Service DNS:

```text
http://service-name.namespace.svc.cluster.local
```

Hoặc nếu cùng namespace:

```text
http://service-name
```

## 16.4 “Running nghĩa là app đã sẵn sàng”

Không hẳn.

Pod Running chỉ nghĩa là container đã chạy. App có thể chưa ready.

Để xác định app sẵn sàng nhận traffic, nên dùng readinessProbe.

## 16.5 “Rollback là quay lại Pod cũ”

Không đúng.

Rollback là quay lại Pod template của revision cũ, rồi Deployment tạo lại Pod theo template đó.

---

# 17. Checklist cần nắm sau bài này

Sau khi học Pod, ReplicaSet và Deployment, cần trả lời được các câu sau:

```text
Pod là gì?
Vì sao Pod IP không bền vững?
ReplicaSet dùng để làm gì?
Deployment khác ReplicaSet thế nào?
Vì sao không nên tạo ReplicaSet trực tiếp?
Deployment quản lý rollout bằng cách nào?
Khi update image, chuyện gì xảy ra phía sau?
Rolling update là gì?
maxSurge và maxUnavailable là gì?
Rollback hoạt động như thế nào?
Selector và label liên quan gì với nhau?
requests và limits khác nhau thế nào?
Làm sao debug Pod bị ImagePullBackOff?
Làm sao debug Pod bị CrashLoopBackOff?
```

Nếu trả lời được các câu này, ta đã nắm khá chắc phần workload stateless cơ bản trong Kubernetes.

---

# 18. Tóm tắt ngắn gọn

Ba khái niệm này có thể nhớ như sau:

```text
Pod = đơn vị nhỏ nhất để chạy container.
ReplicaSet = đảm bảo luôn có đúng số Pod mong muốn.
Deployment = quản lý ReplicaSet, rollout, rollback và version.
```

Luồng hoạt động:

```text
User apply Deployment
   |
   v
Deployment tạo ReplicaSet
   |
   v
ReplicaSet tạo Pod
   |
   v
Pod chạy container
```

Khi scale:

```text
Deployment đổi replicas
ReplicaSet tăng/giảm số Pod
```

Khi update image:

```text
Deployment tạo ReplicaSet mới
Scale up ReplicaSet mới
Scale down ReplicaSet cũ
```

Khi rollback:

```text
Deployment quay lại template của revision cũ
Tạo lại Pod theo version ổn định trước đó
```

Trong thực tế, nếu muốn chạy một API, web server hoặc worker stateless trên Kubernetes, object đầu tiên nên nghĩ tới là **Deployment**.

Pod là nơi container thực sự chạy.

ReplicaSet là người giữ số lượng Pod.

Deployment là người quản lý version và quá trình triển khai an toàn.

Hiểu rõ ba lớp này là nền tảng bắt buộc trước khi học tiếp Service, Ingress, ConfigMap, Secret, PVC, HPA, Helm và CI/CD trên Kubernetes.
