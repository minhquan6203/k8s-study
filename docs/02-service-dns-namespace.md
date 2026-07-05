# Service, DNS và Namespace trong Kubernetes: Cách các ứng dụng tìm thấy nhau trong cluster

Ở bài trước, ta đã học về **Pod**, **ReplicaSet** và **Deployment**. Ta biết rằng Pod là nơi container thực sự chạy, ReplicaSet đảm bảo số lượng Pod mong muốn, còn Deployment quản lý rollout, rollback và version của workload stateless.

Tuy nhiên, còn một vấn đề rất quan trọng:

```text
Pod có IP riêng, nhưng Pod IP không bền vững.
Pod có thể chết, bị xóa, restart, đổi node và được tạo lại với IP mới.
```

Vậy nếu một app khác muốn gọi vào Pod thì gọi bằng cách nào?

Ví dụ có 3 Pod backend:

```text
backend-pod-1   10.244.1.10
backend-pod-2   10.244.2.15
backend-pod-3   10.244.3.20
```

Nếu frontend gọi trực tiếp vào `10.244.1.10`, thì khi Pod đó chết và Pod mới được tạo ra với IP khác, frontend sẽ lỗi.

Để giải quyết vấn đề này, Kubernetes cung cấp object tên là **Service**.

Service cung cấp một địa chỉ ổn định để client gọi vào, còn phía sau Service có thể là một hoặc nhiều Pod thay đổi liên tục.

Ngoài Service, Kubernetes còn có **DNS nội bộ** để các app gọi nhau bằng tên thay vì IP, và **Namespace** để chia tách tài nguyên trong cluster.

Bài này sẽ đi sâu vào ba khái niệm quan trọng:

```text
Service = địa chỉ ổn định để truy cập Pod
DNS = cơ chế gọi service bằng tên
Namespace = không gian logic để chia nhóm tài nguyên
```

---

# 1. Vấn đề: Pod IP không ổn định

Mỗi Pod trong Kubernetes có một IP riêng. Các Pod trong cluster có thể giao tiếp với nhau thông qua Pod IP.

Ví dụ:

```text
web-pod-1   10.244.1.20
api-pod-1   10.244.2.30
```

Pod `web-pod-1` có thể gọi API bằng:

```text
http://10.244.2.30:8080
```

Nhưng cách này không nên dùng trong thực tế.

Lý do là Pod không phải một thực thể bền vững. Pod có thể bị thay thế bất cứ lúc nào:

```text
Node chết
Pod crash
Deployment rollout version mới
Pod bị evict
Scale down rồi scale up
Resource thiếu
Manual delete Pod
```

Khi Pod bị xóa và tạo lại, Pod mới có thể có tên mới và IP mới.

Ví dụ:

```text
api-pod-old   10.244.2.30
```

Sau khi Pod cũ chết:

```text
api-pod-new   10.244.3.55
```

Nếu client vẫn gọi:

```text
http://10.244.2.30:8080
```

thì request sẽ fail.

Vì vậy, nguyên tắc quan trọng là:

```text
Không thiết kế app gọi trực tiếp Pod IP.
Hãy gọi qua Service.
```

---

# 2. Service là gì?

**Service** là Kubernetes object cung cấp một địa chỉ ổn định để truy cập vào một nhóm Pod.

Nói đơn giản:

```text
Pod = instance thật của app
Service = địa chỉ cố định đứng trước các Pod
```

Ví dụ ta có 3 Pod backend:

```text
backend-pod-1   10.244.1.10
backend-pod-2   10.244.2.15
backend-pod-3   10.244.3.20
```

Ta tạo một Service tên `backend-service`.

Service này có IP ổn định trong cluster:

```text
backend-service   10.96.0.50
```

Client chỉ cần gọi:

```text
http://backend-service:8080
```

hoặc:

```text
http://10.96.0.50:8080
```

Kubernetes sẽ route request đến một trong các Pod backend phía sau.

Mô hình tổng quát:

```text
Client
  |
  v
Service backend-service
  |
  +--> backend-pod-1
  +--> backend-pod-2
  +--> backend-pod-3
```

Điểm quan trọng là client không cần biết phía sau có bao nhiêu Pod, Pod tên gì, Pod nằm node nào, Pod IP là gì.

Service sẽ che giấu sự thay đổi của Pod phía sau.

---

# 3. Service chọn Pod bằng label selector

Service không chọn Pod bằng tên Pod cụ thể. Service chọn Pod bằng **label selector**.

Ví dụ các Pod backend có label:

```yaml
labels:
  app: backend
```

Service có selector:

```yaml
selector:
  app: backend
```

Điều đó có nghĩa là:

```text
Service này sẽ route traffic đến các Pod có label app=backend.
```

Ví dụ Deployment backend:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Service tương ứng:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```

Ở đây:

```text
Service selector app=backend
sẽ tìm các Pod có label app=backend
rồi route traffic đến các Pod đó
```

Nếu selector sai, Service sẽ không tìm thấy Pod nào.

Ví dụ Pod có label:

```yaml
labels:
  app: backend
```

Nhưng Service lại ghi:

```yaml
selector:
  app: api
```

Thì Service sẽ không có endpoint nào phía sau.

Đây là lỗi rất hay gặp khi mới học Kubernetes.

---

# 4. port, targetPort và containerPort khác nhau thế nào?

Khi viết Service, ta thường gặp các trường:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Nhiều người mới học rất dễ nhầm giữa `port`, `targetPort` và `containerPort`.

## 4.1 containerPort

`containerPort` nằm trong Pod hoặc Deployment.

Ví dụ:

```yaml
containers:
  - name: backend
    image: my-backend:1.0
    ports:
      - containerPort: 8080
```

Nó mô tả container đang listen ở port 8080.

Nói đơn giản:

```text
containerPort = port mà app trong container đang mở
```

Nếu app FastAPI chạy port 8080, thì containerPort thường là 8080.

## 4.2 port

`port` nằm trong Service.

Ví dụ:

```yaml
ports:
  - port: 80
```

Đây là port mà client dùng để gọi Service.

Nói đơn giản:

```text
port = port của Service
```

Client trong cluster có thể gọi:

```text
http://backend:80
```

hoặc đơn giản:

```text
http://backend
```

nếu là HTTP port 80.

## 4.3 targetPort

`targetPort` là port thật trên Pod mà Service sẽ forward traffic tới.

Ví dụ:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Nghĩa là:

```text
Client gọi Service port 80.
Service forward request đến Pod port 8080.
```

Luồng traffic:

```text
Client
  |
  v
Service backend:80
  |
  v
Pod backend:8080
```

Ví dụ đầy đủ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Nếu app trong container thật sự chạy port 8080, cấu hình này đúng.

Nếu targetPort sai, Service vẫn tồn tại nhưng request sẽ không đến đúng app.

---

# 5. Các loại Service trong Kubernetes

Kubernetes có nhiều loại Service. Mỗi loại phù hợp với một cách expose ứng dụng khác nhau.

Các loại thường gặp:

```text
ClusterIP
NodePort
LoadBalancer
Headless Service
ExternalName
```

Trong bài này, ta tập trung vào 4 loại quan trọng nhất:

```text
ClusterIP
NodePort
LoadBalancer
Headless Service
```

---

# 6. ClusterIP Service

## 6.1 ClusterIP là gì?

**ClusterIP** là loại Service mặc định trong Kubernetes.

Nó tạo ra một IP nội bộ chỉ truy cập được bên trong cluster.

Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Nếu không ghi `type`, Kubernetes mặc định dùng `ClusterIP`.

Tức là hai manifest sau tương đương:

```yaml
spec:
  type: ClusterIP
```

và:

```yaml
spec:
  selector:
    app: backend
```

ClusterIP phù hợp cho service nội bộ.

Ví dụ:

```text
frontend gọi backend
backend gọi user-service
backend gọi payment-service
Spark gọi MinIO nội bộ
Airflow gọi metadata database
```

## 6.2 Khi nào dùng ClusterIP?

Dùng ClusterIP khi service chỉ cần được gọi từ bên trong cluster.

Ví dụ:

```text
frontend-service gọi backend-service
backend-service gọi redis-service
spark job gọi minio-service
```

Mô hình:

```text
Pod A
  |
  v
ClusterIP Service
  |
  v
Pod B
```

ClusterIP không expose app trực tiếp ra bên ngoài cluster.

Nếu muốn người dùng bên ngoài truy cập, thường cần dùng Ingress, NodePort hoặc LoadBalancer.

---

# 7. NodePort Service

## 7.1 NodePort là gì?

**NodePort** expose Service ra ngoài cluster bằng cách mở một port trên mỗi node.

Ví dụ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

Khi đó, client bên ngoài có thể truy cập bằng:

```text
http://<node-ip>:30080
```

Ví dụ:

```text
http://192.168.1.10:30080
```

Traffic đi như sau:

```text
External Client
  |
  v
NodeIP:30080
  |
  v
NodePort Service
  |
  v
backend Pod
```

## 7.2 Đặc điểm của NodePort

NodePort thường nằm trong range mặc định:

```text
30000-32767
```

Khi tạo NodePort, Kubernetes sẽ mở cùng một port trên tất cả node trong cluster.

Ví dụ cluster có 3 node:

```text
node-1   192.168.1.10
node-2   192.168.1.11
node-3   192.168.1.12
```

Service NodePort là `30080`, ta có thể gọi:

```text
http://192.168.1.10:30080
http://192.168.1.11:30080
http://192.168.1.12:30080
```

Kubernetes sẽ route traffic về Pod phù hợp.

## 7.3 Khi nào dùng NodePort?

NodePort thường dùng trong môi trường:

```text
local cluster
bare-metal/on-premise cluster
lab/test nhanh
không có cloud LoadBalancer
muốn expose nhanh một service ra ngoài
```

Ví dụ khi chạy Kubernetes trên máy local, minikube, kind, hoặc cluster nội bộ công ty, NodePort là cách đơn giản để truy cập app từ bên ngoài.

Tuy nhiên, trong production, NodePort thường không phải lựa chọn đẹp nhất nếu expose HTTP/HTTPS cho user cuối. Thường người ta dùng Ingress hoặc LoadBalancer phía trước.

---

# 8. LoadBalancer Service

## 8.1 LoadBalancer là gì?

**LoadBalancer** là loại Service dùng để yêu cầu cloud provider tạo một load balancer bên ngoài cluster.

Ví dụ trên AWS, GCP, Azure, nếu cluster có tích hợp cloud controller, khi tạo Service type LoadBalancer, cloud provider có thể cấp một external IP hoặc DNS.

Manifest:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-lb
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Sau khi tạo, có thể xem:

```bash
kubectl get svc backend-lb
```

Kết quả có thể có external IP:

```text
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
backend-lb   LoadBalancer   10.96.10.20     34.120.10.5      80:31234/TCP
```

Client bên ngoài gọi:

```text
http://34.120.10.5
```

## 8.2 Khi nào dùng LoadBalancer?

Dùng LoadBalancer khi chạy trên cloud và muốn expose service ra internet hoặc mạng ngoài thông qua load balancer của cloud provider.

Ví dụ:

```text
Public API
Frontend service
Gateway service
Ingress Controller service
```

Trong thực tế, nhiều hệ thống không tạo LoadBalancer cho từng app. Thay vào đó, họ tạo một LoadBalancer cho **Ingress Controller**, sau đó dùng Ingress để route nhiều domain/path về nhiều service nội bộ.

Mô hình thường gặp:

```text
Internet
  |
  v
Cloud LoadBalancer
  |
  v
Ingress Controller
  |
  +--> frontend-service
  +--> backend-service
  +--> api-service
```

---

# 9. Headless Service

## 9.1 Headless Service là gì?

Headless Service là Service không có ClusterIP.

Ta khai báo:

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
  ports:
    - port: 9092
      targetPort: 9092
```

Với Service thường, DNS name trả về Service IP.

Với Headless Service, DNS có thể trả về danh sách IP của các Pod phía sau.

Nói đơn giản:

```text
Service thường = có một IP ảo đứng trước các Pod
Headless Service = không có IP ảo, DNS trỏ trực tiếp tới Pod
```

## 9.2 Khi nào dùng Headless Service?

Headless Service thường dùng với StatefulSet hoặc các hệ thống cần biết từng Pod riêng lẻ.

Ví dụ:

```text
Kafka
Zookeeper
Elasticsearch
ScyllaDB
Cassandra
MinIO distributed
Database cluster
```

Các hệ thống này thường cần định danh ổn định cho từng node trong cụm.

Ví dụ StatefulSet Kafka có Pod:

```text
kafka-0
kafka-1
kafka-2
```

Với Headless Service, từng Pod có DNS ổn định dạng:

```text
kafka-0.kafka-headless.namespace.svc.cluster.local
kafka-1.kafka-headless.namespace.svc.cluster.local
kafka-2.kafka-headless.namespace.svc.cluster.local
```

Điều này rất quan trọng cho các hệ thống stateful cần peer discovery.

---

# 10. DNS trong Kubernetes

## 10.1 Vì sao cần DNS?

Nếu chỉ có Service IP, client vẫn phải nhớ IP.

Ví dụ:

```text
backend Service IP = 10.96.0.50
```

Client gọi:

```text
http://10.96.0.50:80
```

Cách này vẫn chưa đẹp vì IP khó nhớ và có thể thay đổi nếu Service bị xóa tạo lại.

Kubernetes cung cấp DNS nội bộ để gọi Service bằng tên.

Ví dụ Service tên `backend`, cùng namespace với client:

```text
http://backend
```

Hoặc đầy đủ hơn:

```text
http://backend.default.svc.cluster.local
```

Đây là cách service discovery phổ biến nhất trong Kubernetes.

---

## 10.2 DNS name đầy đủ của Service

DNS đầy đủ của một Service có dạng:

```text
service-name.namespace.svc.cluster.local
```

Ví dụ Service tên `minio`, namespace `hr-minio`:

```text
minio.hr-minio.svc.cluster.local
```

Nếu MinIO chạy port 9000, client gọi:

```text
http://minio.hr-minio.svc.cluster.local:9000
```

Nếu client nằm cùng namespace với Service, có thể gọi ngắn hơn:

```text
http://minio:9000
```

Nếu client khác namespace, nên dùng ít nhất:

```text
http://minio.hr-minio:9000
```

hoặc đầy đủ:

```text
http://minio.hr-minio.svc.cluster.local:9000
```

Cách nhớ:

```text
Cùng namespace:
service-name

Khác namespace:
service-name.namespace

Đầy đủ:
service-name.namespace.svc.cluster.local
```

---

## 10.3 Ví dụ frontend gọi backend

Giả sử có namespace `demo`.

Backend Service:

```text
backend.demo.svc.cluster.local
```

Frontend nằm cùng namespace `demo`.

Frontend có thể gọi:

```text
http://backend
```

Nếu frontend nằm namespace `web`, còn backend nằm namespace `api`, frontend nên gọi:

```text
http://backend.api
```

hoặc:

```text
http://backend.api.svc.cluster.local
```

---

# 11. CoreDNS là gì?

Trong Kubernetes, DNS nội bộ thường được cung cấp bởi **CoreDNS**.

CoreDNS chạy như một service trong namespace `kube-system`.

Có thể xem bằng:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Hoặc tùy cluster:

```bash
kubectl get pods -n kube-system | grep coredns
```

Khi một Pod cần resolve tên service như:

```text
backend.default.svc.cluster.local
```

Pod sẽ hỏi DNS server nội bộ của cluster. CoreDNS trả về địa chỉ tương ứng.

Nếu DNS lỗi, app có thể không gọi được service bằng tên dù Service và Pod vẫn chạy.

Lỗi DNS thường gặp:

```text
Could not resolve host
Name or service not known
Temporary failure in name resolution
getaddrinfo EAI_AGAIN
```

Khi debug service discovery, cần kiểm tra cả Service, Endpoint và DNS.

---

# 12. kube-proxy làm gì trong Service?

Service là object logic. Để traffic thực sự được route đến Pod, Kubernetes cần cơ chế networking phía dưới.

Một thành phần quan trọng là **kube-proxy**.

kube-proxy chạy trên các node và thiết lập rule mạng để traffic đến Service IP hoặc NodePort được chuyển tới Pod backend phù hợp.

Mô hình đơn giản:

```text
Client
  |
  v
Service IP
  |
  v
kube-proxy rule
  |
  +--> Pod 1
  +--> Pod 2
  +--> Pod 3
```

Tùy cluster, kube-proxy có thể dùng iptables hoặc IPVS.

Với người mới học, chỉ cần hiểu:

```text
Service cung cấp địa chỉ ổn định.
kube-proxy giúp route traffic từ Service đến Pod.
CoreDNS giúp resolve tên Service sang địa chỉ.
```

---

# 13. Endpoint và EndpointSlice

Khi Service chọn được các Pod phù hợp, Kubernetes tạo ra danh sách endpoint phía sau Service.

Endpoint là danh sách IP và port của các Pod mà Service có thể route tới.

Ví dụ Service `backend` chọn được 3 Pod:

```text
10.244.1.10:8080
10.244.2.15:8080
10.244.3.20:8080
```

Ta có thể xem bằng:

```bash
kubectl get endpoints backend
```

Hoặc với Kubernetes mới hơn:

```bash
kubectl get endpointslice -l kubernetes.io/service-name=backend
```

Nếu Service không hoạt động, một trong những bước debug quan trọng là kiểm tra endpoint.

Nếu Service có selector sai, endpoint sẽ rỗng.

Ví dụ:

```text
kubectl get endpoints backend
```

Kết quả:

```text
NAME      ENDPOINTS   AGE
backend   <none>      10m
```

Nghĩa là Service không tìm thấy Pod nào phù hợp.

Nguyên nhân thường là:

```text
Selector không khớp label Pod
Pod chưa Ready
Pod không tồn tại
Service nằm sai namespace
```

---

# 14. Namespace là gì?

## 14.1 Namespace là không gian logic trong cluster

**Namespace** dùng để chia tài nguyên trong Kubernetes thành các nhóm logic.

Ví dụ một cluster có thể có nhiều namespace:

```text
default
kube-system
dev
staging
prod
hr-spark
hr-minio
hr-kafka
monitoring
```

Namespace giúp ta tổ chức tài nguyên theo môi trường, team hoặc hệ thống.

Ví dụ:

```text
Namespace dev: app đang phát triển
Namespace staging: app test trước production
Namespace prod: app production
Namespace monitoring: Prometheus, Grafana
Namespace data: Spark, Kafka, MinIO
```

Khi tạo Pod, Service, Deployment, ConfigMap, Secret, PVC, ta thường tạo chúng trong một namespace cụ thể.

---

## 14.2 Namespace không phải là cluster riêng

Namespace giúp chia logic, nhưng không phải là một cluster vật lý riêng.

Các namespace vẫn dùng chung node, network plugin, control plane và cluster resource.

Ví dụ:

```text
Pod ở namespace dev và Pod ở namespace prod vẫn có thể chạy trên cùng một node.
```

Namespace giúp tổ chức và áp dụng policy, quota, RBAC, nhưng không phải ranh giới bảo mật tuyệt đối nếu không cấu hình thêm NetworkPolicy, RBAC và security policy.

Cần nhớ:

```text
Namespace là logical isolation, không phải physical isolation.
```

---

## 14.3 Tạo Namespace

Manifest:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

Apply:

```bash
kubectl apply -f namespace.yaml
```

Hoặc tạo nhanh:

```bash
kubectl create namespace demo
```

Xem namespace:

```bash
kubectl get namespaces
```

Viết ngắn:

```bash
kubectl get ns
```

---

## 14.4 Làm việc với namespace

Xem Pod trong namespace `demo`:

```bash
kubectl get pods -n demo
```

Apply file vào namespace `demo`:

```bash
kubectl apply -f app.yaml -n demo
```

Xem Service trong namespace `demo`:

```bash
kubectl get svc -n demo
```

Xem tất cả Pod trong mọi namespace:

```bash
kubectl get pods -A
```

hoặc:

```bash
kubectl get pods --all-namespaces
```

Đổi namespace mặc định cho context hiện tại:

```bash
kubectl config set-context --current --namespace=demo
```

Sau đó có thể chạy:

```bash
kubectl get pods
```

mà không cần thêm `-n demo`.

---

# 15. Namespace ảnh hưởng DNS như thế nào?

Namespace là một phần trong DNS name của Service.

DNS đầy đủ:

```text
service-name.namespace.svc.cluster.local
```

Ví dụ:

```text
backend.demo.svc.cluster.local
```

Nếu client và Service cùng namespace, có thể gọi ngắn:

```text
backend
```

Nếu khác namespace, cần gọi kèm namespace:

```text
backend.demo
```

Ví dụ:

```text
frontend ở namespace web
backend ở namespace api
```

Frontend không nên gọi:

```text
http://backend
```

vì DNS sẽ tìm Service `backend` trong namespace `web`.

Frontend nên gọi:

```text
http://backend.api
```

hoặc:

```text
http://backend.api.svc.cluster.local
```

Đây là lỗi rất hay gặp khi chia app ra nhiều namespace.

---

# 16. Manifest mẫu chi tiết

Bây giờ ta viết một ví dụ đầy đủ gồm:

```text
Namespace demo
Deployment backend
Service backend ClusterIP
Deployment client để test DNS
```

## 16.1 Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

## 16.2 Backend Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: demo
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
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

## 16.3 Backend Service ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: demo
  labels:
    app: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Ở đây:

```text
Service name = backend
Namespace = demo
Service DNS = backend.demo.svc.cluster.local
Service port = 80
Pod targetPort = 80
Selector = app=backend
```

Service sẽ route traffic tới các Pod có label `app=backend`.

---

# 17. Chuỗi lệnh thực hành

## 17.1 Apply manifest

Nếu tách file:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f backend-service.yaml
```

Nếu gộp chung một file:

```bash
kubectl apply -f demo.yaml
```

## 17.2 Kiểm tra Deployment, Pod và Service

```bash
kubectl get deploy -n demo
kubectl get pods -n demo -o wide
kubectl get svc -n demo
```

Kỳ vọng:

```text
Deployment backend có 3 replicas
Có 3 Pod nginx running
Có Service backend type ClusterIP
```

## 17.3 Kiểm tra endpoint của Service

```bash
kubectl get endpoints backend -n demo
```

Hoặc:

```bash
kubectl get endpointslice -n demo -l kubernetes.io/service-name=backend
```

Nếu Service hoạt động đúng, endpoint sẽ có IP của các Pod backend.

Ví dụ:

```text
10.244.1.10:80,10.244.2.15:80,10.244.3.20:80
```

Nếu endpoint là `<none>`, cần kiểm tra selector và label.

---

# 18. Test DNS và Service từ trong cluster

Để test Service, ta có thể chạy một Pod tạm thời dùng `curl`.

Ví dụ:

```bash
kubectl run curl-test -n demo --image=curlimages/curl -it --rm -- sh
```

Trong shell của Pod, gọi:

```bash
curl http://backend
```

Vì `curl-test` và `backend` cùng namespace `demo`, gọi `backend` là đủ.

Có thể gọi dạng đầy đủ:

```bash
curl http://backend.demo.svc.cluster.local
```

Nếu có response từ nginx, Service và DNS đang hoạt động.

---

# 19. Test gọi Service khác namespace

Giả sử tạo namespace khác tên `client`:

```bash
kubectl create namespace client
```

Chạy Pod test trong namespace `client`:

```bash
kubectl run curl-test -n client --image=curlimages/curl -it --rm -- sh
```

Nếu gọi:

```bash
curl http://backend
```

có thể fail, vì Kubernetes sẽ tìm Service `backend` trong namespace `client`.

Phải gọi:

```bash
curl http://backend.demo
```

hoặc:

```bash
curl http://backend.demo.svc.cluster.local
```

Đây là ví dụ rất quan trọng để hiểu DNS và namespace.

---

# 20. NodePort manifest mẫu

Nếu muốn expose backend ra ngoài cluster qua NodePort:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
  namespace: demo
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 80
      nodePort: 30080
```

Apply:

```bash
kubectl apply -f backend-nodeport.yaml
```

Xem Service:

```bash
kubectl get svc -n demo
```

Truy cập từ ngoài cluster:

```text
http://<node-ip>:30080
```

Ví dụ:

```text
http://192.168.1.10:30080
```

Nếu đang dùng Docker Desktop, Minikube hoặc Kind, cách lấy node IP có thể khác nhau.

---

# 21. LoadBalancer manifest mẫu

Nếu cluster chạy trên cloud hoặc có load balancer controller hỗ trợ:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-lb
  namespace: demo
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Apply:

```bash
kubectl apply -f backend-lb.yaml
```

Xem external IP:

```bash
kubectl get svc backend-lb -n demo
```

Nếu `EXTERNAL-IP` cứ ở trạng thái `<pending>`, có thể cluster không hỗ trợ LoadBalancer tự động.

Trường hợp on-premise hoặc local cluster, cần dùng giải pháp như MetalLB hoặc dùng NodePort/Ingress tùy môi trường.

---

# 22. Headless Service manifest mẫu

Ví dụ Headless Service:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
  namespace: demo
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
    - name: http
      port: 80
      targetPort: 80
```

Với Headless Service, DNS có thể trả về IP của các Pod phía sau thay vì một ClusterIP duy nhất.

Test:

```bash
kubectl run dns-test -n demo --image=busybox:1.36 -it --rm -- sh
```

Trong Pod:

```bash
nslookup backend-headless
```

Kết quả có thể trả về nhiều IP Pod.

Headless Service thường dùng nhiều hơn với StatefulSet, không phải lựa chọn mặc định cho app stateless thông thường.

---

# 23. So sánh nhanh các loại Service

```text
ClusterIP:
Dùng cho truy cập nội bộ trong cluster.
Đây là loại mặc định.

NodePort:
Expose service ra ngoài qua port trên node.
Phù hợp lab, on-premise, test nhanh.

LoadBalancer:
Yêu cầu cloud provider hoặc load balancer controller cấp external IP.
Phù hợp production trên cloud.

Headless:
Không có ClusterIP.
DNS trỏ trực tiếp tới Pod endpoints.
Phù hợp StatefulSet và service discovery đặc biệt.
```

Cách chọn đơn giản:

```text
App chỉ gọi nội bộ -> ClusterIP
Muốn test từ ngoài cluster nhanh -> NodePort
Muốn expose public trên cloud -> LoadBalancer hoặc Ingress
App stateful cần DNS từng Pod -> Headless Service
```

---

# 24. Troubleshooting Service và DNS

## 24.1 Service không gọi được

Kiểm tra Service:

```bash
kubectl get svc -n demo
kubectl describe svc backend -n demo
```

Kiểm tra Pod:

```bash
kubectl get pods -n demo -o wide
```

Kiểm tra endpoint:

```bash
kubectl get endpoints backend -n demo
```

Nếu endpoint `<none>`, Service không tìm thấy Pod phù hợp.

Nguyên nhân thường gặp:

```text
Service selector sai
Pod label sai
Pod chưa Ready
Service và Pod khác namespace
Pod chưa chạy
```

---

## 24.2 Selector sai

Xem label của Pod:

```bash
kubectl get pods -n demo --show-labels
```

Xem selector của Service:

```bash
kubectl describe svc backend -n demo
```

Nếu Pod label là:

```text
app=backend
```

thì Service selector nên là:

```yaml
selector:
  app: backend
```

Nếu Service selector là:

```yaml
selector:
  app: api
```

thì sẽ không route được.

---

## 24.3 DNS không resolve được

Chạy Pod test:

```bash
kubectl run dns-test -n demo --image=busybox:1.36 -it --rm -- sh
```

Test DNS:

```bash
nslookup backend
nslookup backend.demo.svc.cluster.local
```

Nếu DNS fail, kiểm tra CoreDNS:

```bash
kubectl get pods -n kube-system | grep coredns
```

Kiểm tra Service kube-dns:

```bash
kubectl get svc -n kube-system
```

Lỗi thường gặp:

```text
CoreDNS Pod lỗi
Network plugin lỗi
Pod DNS config lỗi
Gọi sai namespace
Service không tồn tại
```

---

## 24.4 Gọi sai namespace

Giả sử Service nằm ở namespace `demo`.

Nếu Pod client cũng ở namespace `demo`, gọi được:

```text
http://backend
```

Nếu Pod client ở namespace khác, phải gọi:

```text
http://backend.demo
```

hoặc:

```text
http://backend.demo.svc.cluster.local
```

Nếu không thêm namespace, DNS sẽ tìm service trong namespace hiện tại của Pod client.

---

## 24.5 targetPort sai

Nếu Service có endpoint nhưng gọi vẫn không được, kiểm tra app thật sự listen port nào.

Ví dụ container chạy port 8080, nhưng Service ghi:

```yaml
targetPort: 80
```

thì request sẽ không đến đúng app.

Cần sửa:

```yaml
targetPort: 8080
```

Kiểm tra port trong Deployment:

```yaml
containers:
  - name: backend
    ports:
      - containerPort: 8080
```

Và kiểm tra app config bên trong container.

---

## 24.6 NodePort không truy cập được từ ngoài

Kiểm tra Service:

```bash
kubectl get svc -n demo
```

Kiểm tra node IP:

```bash
kubectl get nodes -o wide
```

Kiểm tra firewall/security group:

```text
Port NodePort có được mở không?
Client có route được tới node IP không?
Cluster chạy local hay VM?
Service có endpoint không?
```

Nếu Service NodePort có endpoint đúng nhưng bên ngoài vẫn không vào được, thường là do network/firewall/security group.

---

# 25. Bài lab tự làm

## Lab 1: Tạo namespace demo

Tạo namespace:

```bash
kubectl create namespace demo
```

Kiểm tra:

```bash
kubectl get ns
```

## Lab 2: Deploy backend 3 replicas

Tạo Deployment `backend` trong namespace `demo`.

Kiểm tra:

```bash
kubectl get deploy,pods -n demo -o wide
```

Yêu cầu:

```text
Có 3 Pod backend running
Các Pod có label app=backend
```

## Lab 3: Tạo ClusterIP Service

Tạo Service `backend` type ClusterIP.

Kiểm tra:

```bash
kubectl get svc -n demo
kubectl get endpoints backend -n demo
```

Yêu cầu:

```text
Service backend có ClusterIP
Endpoint trỏ đến IP của 3 Pod backend
```

## Lab 4: Test DNS cùng namespace

Chạy Pod curl trong namespace demo:

```bash
kubectl run curl-test -n demo --image=curlimages/curl -it --rm -- sh
```

Gọi:

```bash
curl http://backend
curl http://backend.demo.svc.cluster.local
```

Yêu cầu:

```text
Cả hai cách đều gọi được backend
```

## Lab 5: Test DNS khác namespace

Tạo namespace client:

```bash
kubectl create namespace client
```

Chạy Pod curl:

```bash
kubectl run curl-test -n client --image=curlimages/curl -it --rm -- sh
```

Thử gọi:

```bash
curl http://backend
```

Sau đó gọi đúng:

```bash
curl http://backend.demo
```

Yêu cầu hiểu được:

```text
backend fail vì tìm service trong namespace client
backend.demo đúng vì chỉ rõ namespace demo
```

## Lab 6: Làm sai selector để quan sát lỗi

Sửa Service selector thành:

```yaml
selector:
  app: wrong
```

Apply lại.

Kiểm tra:

```bash
kubectl get endpoints backend -n demo
```

Yêu cầu:

```text
Endpoint rỗng
Service không route được đến Pod
```

Sau đó sửa lại:

```yaml
selector:
  app: backend
```

## Lab 7: Tạo NodePort Service

Tạo Service NodePort:

```yaml
type: NodePort
nodePort: 30080
```

Truy cập từ ngoài:

```text
http://<node-ip>:30080
```

Yêu cầu hiểu được:

```text
NodePort expose app ra ngoài thông qua port trên node
```

---

# 26. Checklist cần nắm sau bài này

Sau khi học Service, DNS và Namespace, cần trả lời được:

```text
Vì sao không nên gọi trực tiếp Pod IP?
Service là gì?
Service chọn Pod bằng cách nào?
selector và label liên quan gì với nhau?
port, targetPort và containerPort khác nhau thế nào?
ClusterIP dùng khi nào?
NodePort dùng khi nào?
LoadBalancer dùng khi nào?
Headless Service dùng khi nào?
DNS đầy đủ của một Service có dạng gì?
Khi nào gọi được bằng service-name?
Khi nào phải gọi service-name.namespace?
Namespace là gì?
Namespace có phải cluster riêng không?
CoreDNS dùng để làm gì?
Endpoint/EndpointSlice là gì?
Service có endpoint rỗng thì lỗi thường nằm ở đâu?
```

---

# 27. Tóm tắt ngắn gọn

Ba khái niệm chính:

```text
Service = địa chỉ ổn định để truy cập nhóm Pod
DNS = gọi Service bằng tên thay vì IP
Namespace = chia tài nguyên trong cluster theo không gian logic
```

Luồng traffic cơ bản:

```text
Client Pod
  |
  v
Service DNS name
  |
  v
Service ClusterIP
  |
  v
Endpoint Pod IPs
  |
  v
Backend Pods
```

Cách Service tìm Pod:

```text
Service selector
  |
  v
Pod labels
```

Ví dụ:

```yaml
selector:
  app: backend
```

sẽ chọn Pod có:

```yaml
labels:
  app: backend
```

DNS Service:

```text
service-name.namespace.svc.cluster.local
```

Ví dụ:

```text
backend.demo.svc.cluster.local
```

Nếu cùng namespace:

```text
backend
```

Nếu khác namespace:

```text
backend.demo
```

Các loại Service cần nhớ:

```text
ClusterIP = nội bộ cluster
NodePort = expose qua port trên node
LoadBalancer = expose qua load balancer bên ngoài
Headless = DNS trỏ trực tiếp đến Pod, hay dùng cho StatefulSet
```

