# ConfigMap và Secret trong Kubernetes: Quản lý cấu hình và thông tin nhạy cảm cho ứng dụng

Ở các bài trước, ta đã học về **Pod, ReplicaSet, Deployment**, sau đó là **Service, DNS và Namespace**.

Đến đây, ta đã biết cách chạy một ứng dụng stateless trên Kubernetes và biết cách để các ứng dụng gọi nhau thông qua Service. Tuy nhiên, một ứng dụng thực tế không chỉ có image và port. Nó còn cần rất nhiều cấu hình để chạy đúng.

Ví dụ một backend API có thể cần:

```text id="a2t8wp"
APP_ENV=prod
LOG_LEVEL=info
DATABASE_HOST=postgres.default.svc.cluster.local
DATABASE_PORT=5432
REDIS_HOST=redis.default.svc.cluster.local
S3_ENDPOINT=http://minio.hr-minio.svc.cluster.local:9000
```

Ngoài ra, app còn có thể cần thông tin nhạy cảm:

```text id="k67f9z"
DATABASE_USERNAME
DATABASE_PASSWORD
JWT_SECRET
ACCESS_KEY
SECRET_KEY
API_TOKEN
```

Nếu nhét trực tiếp toàn bộ các giá trị này vào Docker image hoặc hard-code trong source code, hệ thống sẽ rất khó maintain và kém an toàn.

Kubernetes giải quyết vấn đề này bằng hai object quan trọng:

```text id="imxxpu"
ConfigMap = lưu cấu hình không nhạy cảm
Secret = lưu thông tin nhạy cảm
```

Theo Kubernetes documentation, ConfigMap được dùng để lưu dữ liệu không nhạy cảm dạng key-value, và Pod có thể dùng ConfigMap dưới dạng environment variables, command-line arguments hoặc file cấu hình được mount qua volume. ConfigMap giúp tách cấu hình theo môi trường ra khỏi container image.

Secret cũng có thể được mount vào Pod dưới dạng volume hoặc expose thành environment variables, nhưng được dùng cho dữ liệu nhạy cảm như password, token, key hoặc credential.

---

# 1. Vì sao cần ConfigMap và Secret?

Một ứng dụng thường chạy ở nhiều môi trường khác nhau:

```text id="xl0qqd"
local
dev
staging
production
```

Cùng một image app, nhưng cấu hình mỗi môi trường có thể khác nhau.

Ví dụ:

```text id="mx7rjk"
dev:
DATABASE_HOST=postgres-dev
LOG_LEVEL=debug

prod:
DATABASE_HOST=postgres-prod
LOG_LEVEL=info
```

Nếu cấu hình bị hard-code trong source code, mỗi lần đổi môi trường ta phải sửa code, build lại image, deploy lại app. Cách này không linh hoạt.

Nếu cấu hình bị nhét trực tiếp vào Docker image, ta phải build nhiều image khác nhau:

```text id="u2xd2z"
my-api:dev
my-api:staging
my-api:prod
```

Điều này không tốt vì image nên đại diện cho version của app, không nên bị gắn chặt với môi trường.

Cách tốt hơn là:

```text id="aen0zj"
Image chứa code và dependency.
ConfigMap/Secret chứa cấu hình runtime.
Deployment gắn config vào Pod khi chạy.
```

Mô hình đúng hơn:

```text id="bk5m0j"
Docker image:
- source code
- runtime
- dependencies

ConfigMap:
- APP_ENV
- LOG_LEVEL
- DATABASE_HOST
- FEATURE_FLAG

Secret:
- DATABASE_PASSWORD
- ACCESS_TOKEN
- JWT_SECRET
```

Khi deploy sang môi trường khác, ta giữ nguyên image nhưng thay ConfigMap/Secret.

---

# 2. ConfigMap là gì?

**ConfigMap** là Kubernetes object dùng để lưu cấu hình không nhạy cảm dạng key-value.

Ví dụ:

```yaml id="k0g03m"
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:
  APP_ENV: "dev"
  LOG_LEVEL: "debug"
  DATABASE_HOST: "postgres.demo.svc.cluster.local"
  DATABASE_PORT: "5432"
```

Ở đây, ConfigMap tên `app-config` lưu 4 cấu hình:

```text id="8dzg1r"
APP_ENV=dev
LOG_LEVEL=debug
DATABASE_HOST=postgres.demo.svc.cluster.local
DATABASE_PORT=5432
```

ConfigMap phù hợp cho các dữ liệu như:

```text id="yfswpn"
Tên môi trường
Log level
URL service nội bộ
Database host
Feature flag
Timeout
Batch size
Tên bucket
Tên topic Kafka
Tên index Elasticsearch
```

ConfigMap không phù hợp cho:

```text id="70bb60"
Password
Token
Private key
Secret key
Credential
Certificate private material
```

Kubernetes documentation cũng nhấn mạnh rằng ConfigMap không cung cấp secrecy hoặc encryption, nên dữ liệu confidential nên dùng Secret hoặc công cụ bổ sung để bảo vệ dữ liệu.

---

# 3. Secret là gì?

**Secret** là Kubernetes object dùng để lưu thông tin nhạy cảm.

Ví dụ:

```yaml id="20l7wv"
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
stringData:
  DATABASE_USERNAME: "app_user"
  DATABASE_PASSWORD: "super_password"
  JWT_SECRET: "my_jwt_secret"
```

Secret phù hợp cho:

```text id="r9v9vb"
Database username/password
API token
JWT secret
S3 access key/secret key
Elasticsearch password
Private registry credential
TLS certificate
SSH key
```

Secret có thể được dùng bởi Pod dưới dạng environment variables hoặc file được mount qua volume. Kubernetes cũng yêu cầu Secret phải tồn tại trước khi Pod phụ thuộc vào Secret đó được chạy, trừ khi Secret được khai báo optional.

---

# 4. ConfigMap khác Secret thế nào?

Nhìn đơn giản:

```text id="37fhtg"
ConfigMap = config thường
Secret = config nhạy cảm
```

So sánh:

```text id="vr7421"
ConfigMap:
- Lưu cấu hình không nhạy cảm
- Dạng key-value
- Có thể inject vào Pod qua env hoặc volume
- Không dùng cho password/token

Secret:
- Lưu thông tin nhạy cảm
- Dạng key-value
- Có thể inject vào Pod qua env hoặc volume
- Dùng cho password/token/key/cert
```

Ví dụ cùng một app:

```text id="k3y2l5"
ConfigMap:
APP_ENV=prod
LOG_LEVEL=info
DATABASE_HOST=postgres.prod.svc.cluster.local
DATABASE_PORT=5432

Secret:
DATABASE_USERNAME=app_user
DATABASE_PASSWORD=xxxxx
JWT_SECRET=xxxxx
```

Một rule dễ nhớ:

```text id="ggc0am"
Nếu lộ ra mà không sao nhiều -> ConfigMap.
Nếu lộ ra có thể gây rủi ro bảo mật -> Secret.
```

---

# 5. Cách Pod sử dụng ConfigMap và Secret

Pod thường sử dụng ConfigMap/Secret theo hai cách chính:

```text id="qqq0qh"
1. Inject thành environment variables
2. Mount thành file thông qua volume
```

Theo Kubernetes docs, có thể set biến môi trường cho container bằng `env` hoặc `envFrom`; `env` set từng biến cụ thể, còn `envFrom` lấy toàn bộ key-value từ ConfigMap hoặc Secret làm environment variables.

---

# 6. Cách 1: Dùng ConfigMap làm environment variables

## 6.1 Dùng từng key bằng configMapKeyRef

Ví dụ ConfigMap:

```yaml id="cxwqpp"
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:
  APP_ENV: "dev"
  LOG_LEVEL: "debug"
  DATABASE_HOST: "postgres.demo.svc.cluster.local"
  DATABASE_PORT: "5432"
```

Deployment dùng từng key:

```yaml id="6v604c"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: LOG_LEVEL
            - name: DATABASE_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DATABASE_HOST
            - name: DATABASE_PORT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DATABASE_PORT
```

Ý nghĩa:

```text id="6it6r8"
Lấy key APP_ENV từ ConfigMap app-config, đưa vào env APP_ENV trong container.
Lấy key LOG_LEVEL từ ConfigMap app-config, đưa vào env LOG_LEVEL trong container.
```

Cách này rõ ràng, kiểm soát được từng biến nào được inject vào container.

---

## 6.2 Dùng toàn bộ ConfigMap bằng envFrom

Nếu muốn lấy toàn bộ key-value trong ConfigMap làm environment variables:

```yaml id="kh91d6"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          envFrom:
            - configMapRef:
                name: app-config
```

Với ConfigMap:

```yaml id="z2jicz"
data:
  APP_ENV: "dev"
  LOG_LEVEL: "debug"
  DATABASE_HOST: "postgres.demo.svc.cluster.local"
  DATABASE_PORT: "5432"
```

Container sẽ có các biến môi trường:

```text id="fj4pn9"
APP_ENV=dev
LOG_LEVEL=debug
DATABASE_HOST=postgres.demo.svc.cluster.local
DATABASE_PORT=5432
```

Ưu điểm:

```text id="wu5zk3"
Manifest ngắn
Dễ inject nhiều biến
Phù hợp config đơn giản
```

Nhược điểm:

```text id="hf578d"
Khó kiểm soát từng biến
Nếu ConfigMap có key không mong muốn, container cũng nhận luôn
Tên key phải hợp lệ để trở thành environment variable
```

Cách dùng `envFrom` thường tiện trong lab hoặc app nhỏ, nhưng trong production nhiều team thích dùng `env` rõ từng biến quan trọng.

---

# 7. Cách 2: Dùng Secret làm environment variables

Secret cũng có thể inject vào Pod giống ConfigMap.

Ví dụ Secret:

```yaml id="y965ur"
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
stringData:
  DATABASE_USERNAME: "app_user"
  DATABASE_PASSWORD: "super_password"
```

Deployment dùng Secret:

```yaml id="nym9u1"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.27
          env:
            - name: DATABASE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: DATABASE_USERNAME
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: DATABASE_PASSWORD
```

Ý nghĩa:

```text id="gygnd2"
Lấy DATABASE_USERNAME từ Secret app-secret.
Lấy DATABASE_PASSWORD từ Secret app-secret.
Inject vào container dưới dạng environment variables.
```

Hoặc dùng `envFrom`:

```yaml id="ay7ppw"
envFrom:
  - secretRef:
      name: app-secret
```

Khi đó toàn bộ key trong Secret sẽ được đưa thành environment variables.

---

# 8. Cách 3: Mount ConfigMap thành file

Ngoài environment variables, ConfigMap có thể được mount vào container dưới dạng file.

Cách này phù hợp khi app đọc config từ file, ví dụ:

```text id="boepph"
application.properties
application.yaml
nginx.conf
log4j.properties
spark-defaults.conf
config.json
```

Ví dụ ConfigMap chứa file config:

```yaml id="vw1biv"
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: demo
data:
  default.conf: |
    server {
      listen 80;
      server_name localhost;

      location / {
        return 200 "Hello from ConfigMap mounted file\n";
      }
    }
```

Deployment mount ConfigMap vào container:

```yaml id="hqv9mj"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-custom
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-custom
  template:
    metadata:
      labels:
        app: nginx-custom
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config-volume
              mountPath: /etc/nginx/conf.d
              readOnly: true
      volumes:
        - name: nginx-config-volume
          configMap:
            name: nginx-config
```

Ở đây:

```text id="9uqx2l"
ConfigMap nginx-config có key default.conf.
Kubernetes mount key đó thành file /etc/nginx/conf.d/default.conf.
Nginx đọc config từ file này.
```

Luồng:

```text id="1n5hgi"
ConfigMap data.default.conf
   |
   v
Mounted file trong container
   |
   v
/etc/nginx/conf.d/default.conf
```

---

# 9. Cách 4: Mount Secret thành file

Secret cũng có thể được mount thành file.

Ví dụ Secret:

```yaml id="xpay6h"
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
stringData:
  username: "app_user"
  password: "super_password"
```

Pod mount Secret:

```yaml id="2fxxee"
apiVersion: v1
kind: Pod
metadata:
  name: secret-file-demo
  namespace: demo
spec:
  containers:
    - name: busybox
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/app-secret
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: app-secret
```

Trong container sẽ có:

```text id="mp7t67"
/etc/app-secret/username
/etc/app-secret/password
```

Có thể kiểm tra:

```bash id="847l24"
kubectl exec -n demo secret-file-demo -- cat /etc/app-secret/username
kubectl exec -n demo secret-file-demo -- cat /etc/app-secret/password
```

Mount Secret thành file thường phù hợp với:

```text id="iaabxy"
TLS certificate
Private key
Credential file
Cloud provider credential
App cần đọc secret từ file path
```

---

# 10. env vs volume mount: nên dùng cái nào?

Cả ConfigMap và Secret đều có thể dùng qua env hoặc volume. Mỗi cách có ưu/nhược điểm.

## 10.1 Dùng environment variables

Phù hợp khi app đọc config từ env:

```text id="hv13p2"
DATABASE_HOST
DATABASE_PORT
LOG_LEVEL
APP_ENV
TOKEN
```

Ưu điểm:

```text id="m8plbg"
Dễ dùng
Phổ biến với app cloud-native
Dễ debug bằng printenv
Manifest dễ đọc
```

Nhược điểm:

```text id="jw1y3y"
Khi ConfigMap/Secret thay đổi, env trong container đang chạy không tự đổi
Cần restart Pod để nhận giá trị mới
Secret dưới dạng env có thể dễ bị lộ qua debug/log/process environment nếu thao tác không cẩn thận
```

## 10.2 Dùng volume mount

Phù hợp khi app đọc config từ file:

```text id="dwb05o"
nginx.conf
application.yaml
config.json
TLS cert/key
spark-defaults.conf
```

Ưu điểm:

```text id="9swej7"
Phù hợp với file config dài
Có thể mount nhiều file
Một số update có thể được phản ánh vào file được mount sau một khoảng trễ
Phù hợp với certificate/key file
```

Nhược điểm:

```text id="5vxm2l"
App phải biết đọc config từ file
Nếu app chỉ đọc config lúc startup, vẫn cần restart hoặc reload app
Mount sai path có thể ghi đè nội dung thư mục
Nếu dùng subPath, update tự động có thể không được phản ánh
```

Kubernetes documentation ghi rõ container dùng Secret dưới dạng `subPath` volume mount sẽ không nhận được Secret updates tự động. ConfigMap mounted vào Pod cũng có cơ chế propagation từ kubelet cache/watch và có thể có độ trễ trước khi key mới được project vào Pod.

Cách nhớ thực tế:

```text id="dtyi0u"
Config ngắn, app đọc env -> dùng env.
Config dạng file, cert, key, nginx config -> dùng volume mount.
Đổi config mà app không tự reload -> restart Pod.
```

---

# 11. data vs stringData trong Secret

Khi viết Secret, ta thường thấy hai field:

```text id="mecfnf"
data
stringData
```

## 11.1 data

Với `data`, giá trị phải được base64 encode.

Ví dụ muốn lưu:

```text id="xsre9m"
DATABASE_PASSWORD=super_password
```

Encode base64:

```bash id="vdydwb"
echo -n "super_password" | base64
```

Kết quả ví dụ:

```text id="lom0gu"
c3VwZXJfcGFzc3dvcmQ=
```

Secret:

```yaml id="mwtos1"
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
data:
  DATABASE_PASSWORD: c3VwZXJfcGFzc3dvcmQ=
```

## 11.2 stringData

Với `stringData`, ta có thể ghi plain text trong manifest, Kubernetes sẽ xử lý chuyển sang `data`.

Ví dụ:

```yaml id="sx844x"
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
stringData:
  DATABASE_PASSWORD: "super_password"
```

Trong lab, `stringData` dễ đọc và dễ viết hơn.

Tuy nhiên, cần hiểu rằng nếu commit file Secret plain text vào Git thì vẫn bị lộ. Secret trong Kubernetes không có nghĩa là file YAML của m tự nhiên an toàn.

---

# 12. Base64 không phải mã hóa bảo mật

Một hiểu nhầm rất phổ biến là:

```text id="8ue5tr"
Secret dùng base64 nên an toàn.
```

Không đúng.

Base64 chỉ là encoding, không phải encryption.

Ví dụ:

```bash id="kks8am"
echo "c3VwZXJfcGFzc3dvcmQ=" | base64 -d
```

Sẽ ra:

```text id="jwbmcm"
super_password
```

Vì vậy, không nên nghĩ rằng Secret YAML có base64 là đủ an toàn.

Cần nhớ:

```text id="p69hit"
Base64 chỉ giúp biểu diễn dữ liệu dạng text.
Base64 không bảo vệ secret khỏi người có quyền đọc.
Ai đọc được Secret thì decode được nội dung.
```

Trong production, cần quan tâm thêm:

```text id="7b1vpg"
RBAC: ai được get/list/watch Secret
Encryption at rest cho etcd
External Secret Manager
Không commit Secret thật vào Git
Audit log
Least privilege ServiceAccount
```

---

# 13. Manifest mẫu đầy đủ

Bây giờ ta viết một ví dụ đầy đủ gồm:

```text id="jpqmcl"
Namespace demo
ConfigMap app-config
Secret app-secret
Deployment app dùng cả ConfigMap và Secret
Service để test
```

## 13.1 Namespace

```yaml id="wbux4x"
apiVersion: v1
kind: Namespace
metadata:
  name: demo
```

## 13.2 ConfigMap

```yaml id="33lri8"
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: demo
data:
  APP_ENV: "dev"
  LOG_LEVEL: "debug"
  APP_MESSAGE: "Hello from ConfigMap"
```

## 13.3 Secret

```yaml id="sxhy1l"
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: demo
type: Opaque
stringData:
  APP_USERNAME: "demo_user"
  APP_PASSWORD: "demo_password"
```

## 13.4 Deployment dùng ConfigMap và Secret qua env

Dùng image `busybox` để dễ test biến môi trường:

```yaml id="5rhjxq"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-secret-demo
  namespace: demo
  labels:
    app: config-secret-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-secret-demo
  template:
    metadata:
      labels:
        app: config-secret-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo APP_ENV=$APP_ENV LOG_LEVEL=$LOG_LEVEL APP_MESSAGE=$APP_MESSAGE; sleep 30; done"]
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_ENV
            - name: LOG_LEVEL
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: LOG_LEVEL
            - name: APP_MESSAGE
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_MESSAGE
            - name: APP_USERNAME
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: APP_USERNAME
            - name: APP_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secret
                  key: APP_PASSWORD
```

Apply:

```bash id="2373d3"
kubectl apply -f demo.yaml
```

Xem log:

```bash id="pgq3do"
kubectl logs -n demo deploy/config-secret-demo
```

Exec vào Pod để xem env:

```bash id="ul6a4w"
kubectl exec -n demo deploy/config-secret-demo -- printenv | grep APP
```

Kỳ vọng thấy:

```text id="hc2p82"
APP_ENV=dev
LOG_LEVEL=debug
APP_MESSAGE=Hello from ConfigMap
APP_USERNAME=demo_user
APP_PASSWORD=demo_password
```

---

# 14. Manifest dùng envFrom

Nếu muốn ngắn hơn, có thể dùng `envFrom`.

```yaml id="al078s"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: envfrom-demo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: envfrom-demo
  template:
    metadata:
      labels:
        app: envfrom-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "printenv | sort && sleep 3600"]
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secret
```

Với cách này, toàn bộ key trong `app-config` và `app-secret` sẽ thành environment variables trong container.

Ưu điểm là YAML ngắn hơn. Nhược điểm là container nhận toàn bộ key, đôi khi không muốn expose hết như vậy.

---

# 15. Manifest mount ConfigMap và Secret thành file

Ví dụ ConfigMap chứa file `app.properties`:

```yaml id="kkh8wv"
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-file-config
  namespace: demo
data:
  app.properties: |
    app.env=dev
    log.level=debug
    feature.enabled=true
```

Secret chứa file credential:

```yaml id="0oe4w2"
apiVersion: v1
kind: Secret
metadata:
  name: app-file-secret
  namespace: demo
type: Opaque
stringData:
  username: "demo_user"
  password: "demo_password"
```

Pod mount cả hai:

```yaml id="0jb2fg"
apiVersion: v1
kind: Pod
metadata:
  name: file-mount-demo
  namespace: demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app-config
          readOnly: true
        - name: secret-volume
          mountPath: /etc/app-secret
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-file-config
    - name: secret-volume
      secret:
        secretName: app-file-secret
```

Kiểm tra file:

```bash id="qh5oy6"
kubectl exec -n demo file-mount-demo -- ls -l /etc/app-config
kubectl exec -n demo file-mount-demo -- cat /etc/app-config/app.properties
kubectl exec -n demo file-mount-demo -- ls -l /etc/app-secret
kubectl exec -n demo file-mount-demo -- cat /etc/app-secret/username
kubectl exec -n demo file-mount-demo -- cat /etc/app-secret/password
```

Kết quả:

```text id="9h1gba"
/etc/app-config/app.properties
/etc/app-secret/username
/etc/app-secret/password
```

---

# 16. Chuỗi lệnh thực hành

## 16.1 Tạo namespace

```bash id="lpaz9f"
kubectl create namespace demo
```

Hoặc apply YAML:

```bash id="lt9qf9"
kubectl apply -f namespace.yaml
```

## 16.2 Tạo ConfigMap từ YAML

```bash id="a52nle"
kubectl apply -f app-config.yaml
```

Xem ConfigMap:

```bash id="a4fch7"
kubectl get configmap -n demo
kubectl describe configmap app-config -n demo
```

Viết ngắn:

```bash id="h3i2nz"
kubectl get cm -n demo
kubectl describe cm app-config -n demo
```

Xem YAML:

```bash id="oaem20"
kubectl get cm app-config -n demo -o yaml
```

## 16.3 Tạo ConfigMap bằng command

Có thể tạo ConfigMap trực tiếp từ literal:

```bash id="1nd749"
kubectl create configmap app-config \
  -n demo \
  --from-literal=APP_ENV=dev \
  --from-literal=LOG_LEVEL=debug
```

Tạo ConfigMap từ file:

```bash id="3qzhhy"
kubectl create configmap nginx-config \
  -n demo \
  --from-file=default.conf
```

## 16.4 Tạo Secret từ YAML

```bash id="15npzc"
kubectl apply -f app-secret.yaml
```

Xem Secret:

```bash id="m59mmp"
kubectl get secret -n demo
kubectl describe secret app-secret -n demo
```

Lưu ý: `describe secret` không hiện plaintext value.

Xem YAML:

```bash id="xn5d9z"
kubectl get secret app-secret -n demo -o yaml
```

Giá trị trong `data` sẽ ở dạng base64.

## 16.5 Tạo Secret bằng command

```bash id="h7jm1j"
kubectl create secret generic app-secret \
  -n demo \
  --from-literal=APP_USERNAME=demo_user \
  --from-literal=APP_PASSWORD=demo_password
```

Tạo Secret từ file:

```bash id="wn1cii"
kubectl create secret generic app-file-secret \
  -n demo \
  --from-file=username.txt \
  --from-file=password.txt
```

## 16.6 Decode Secret để kiểm tra

```bash id="ufalje"
kubectl get secret app-secret -n demo -o jsonpath='{.data.APP_PASSWORD}' | base64 -d
```

Lưu ý: lệnh này chỉ dùng để lab/debug. Không nên in secret ra terminal/log bừa bãi trong môi trường thật.

---

# 17. Update ConfigMap/Secret thì Pod có tự nhận không?

Đây là phần rất quan trọng khi vận hành thực tế.

## 17.1 Nếu dùng env

Nếu Pod nhận ConfigMap/Secret qua environment variables, thì khi ConfigMap/Secret thay đổi, biến môi trường trong container đang chạy không tự thay đổi.

Ví dụ ban đầu:

```text id="kgr8me"
APP_ENV=dev
```

Update ConfigMap thành:

```text id="uj8745"
APP_ENV=prod
```

Pod đang chạy vẫn giữ env cũ.

Muốn Pod nhận config mới, thường cần restart Pod hoặc rollout Deployment:

```bash id="4zbb84"
kubectl rollout restart deploy/config-secret-demo -n demo
```

## 17.2 Nếu dùng volume mount

Nếu ConfigMap/Secret được mount thành file, Kubernetes có thể cập nhật file trong Pod sau một khoảng trễ, tùy cơ chế watch/cache/sync của kubelet. Tuy nhiên, app có tự đọc lại file hay không là chuyện khác.

Có app chỉ đọc config lúc startup. Với app đó, dù file đã đổi, app vẫn không dùng config mới cho đến khi reload hoặc restart.

Vì vậy, trong thực tế:

```text id="2mabz9"
Env thay đổi -> restart Pod.
File config thay đổi -> có thể cần reload app hoặc restart Pod.
```

Cách phổ biến, dễ kiểm soát nhất:

```bash id="ifepu1"
kubectl rollout restart deployment/<name> -n <namespace>
```

Ví dụ:

```bash id="l1bvvb"
kubectl rollout restart deploy/config-secret-demo -n demo
```

---

# 18. Immutable ConfigMap và Secret

Trong một số trường hợp, ta không muốn ConfigMap/Secret bị sửa sau khi tạo.

Kubernetes hỗ trợ field:

```yaml id="8afazt"
immutable: true
```

Ví dụ ConfigMap immutable:

```yaml id="4ibypd"
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v1
  namespace: demo
immutable: true
data:
  APP_ENV: "prod"
  LOG_LEVEL: "info"
```

Secret immutable:

```yaml id="vcolaj"
apiVersion: v1
kind: Secret
metadata:
  name: app-secret-v1
  namespace: demo
immutable: true
type: Opaque
stringData:
  API_TOKEN: "token-value"
```

Khi đã immutable, không thể sửa data của object đó. Nếu muốn đổi config, tạo object mới, ví dụ:

```text id="gylu0j"
app-config-v1
app-config-v2
app-config-v3
```

Sau đó update Deployment tham chiếu sang ConfigMap mới.

Cách này giúp config version rõ hơn, tránh việc ai đó sửa config đang được dùng mà không kiểm soát.

---

# 19. Best practices khi dùng ConfigMap

## 19.1 Không lưu secret trong ConfigMap

Không lưu các giá trị như:

```text id="7rng0y"
password
token
secret key
access key
private key
```

vào ConfigMap.

Dùng Secret hoặc external secret manager.

## 19.2 Tách config theo môi trường

Ví dụ:

```text id="34xqdx"
app-config-dev
app-config-staging
app-config-prod
```

Hoặc cùng tên `app-config` nhưng nằm ở các namespace khác nhau:

```text id="f3f7cm"
dev/app-config
staging/app-config
prod/app-config
```

Cách dùng namespace thường sạch hơn:

```text id="s7is7d"
namespace dev có app-config riêng
namespace prod có app-config riêng
```

## 19.3 Tránh ConfigMap quá lớn

ConfigMap không nên dùng để lưu dữ liệu lớn.

Nên dùng ConfigMap cho config nhỏ, không phải dataset, file lớn, model, hoặc binary nặng.

## 19.4 Đặt tên key rõ ràng

Nên đặt tên dễ hiểu:

```text id="h1wzsp"
DATABASE_HOST
DATABASE_PORT
LOG_LEVEL
APP_ENV
KAFKA_BOOTSTRAP_SERVERS
S3_ENDPOINT
```

Tránh tên mơ hồ:

```text id="2xvvlv"
HOST
URL
VALUE
CONFIG
```

## 19.5 Dùng rollout restart sau khi đổi config env

Nếu app nhận config qua env, sau khi update ConfigMap nên restart Deployment:

```bash id="lb6sdo"
kubectl rollout restart deploy/<deployment-name> -n <namespace>
```

---

# 20. Best practices khi dùng Secret

## 20.1 Không commit Secret thật vào Git

Dù dùng `data` base64 hay `stringData`, nếu file Secret chứa secret thật và bị commit lên Git thì vẫn bị lộ.

Không nên commit file kiểu:

```yaml id="1v7mj2"
stringData:
  DATABASE_PASSWORD: "real-prod-password"
```

## 20.2 Giới hạn quyền đọc Secret bằng RBAC

Không phải ai cũng nên có quyền:

```text id="mt9wko"
get secrets
list secrets
watch secrets
```

Vì ai có quyền đọc Secret thì có thể decode được dữ liệu.

Nên áp dụng least privilege.

## 20.3 Dùng external secret manager nếu production nghiêm túc

Trong production, có thể dùng các giải pháp như:

```text id="9q3117"
HashiCorp Vault
AWS Secrets Manager
GCP Secret Manager
Azure Key Vault
External Secrets Operator
Sealed Secrets
SOPS
```

Mục tiêu là không để secret plain text nằm trực tiếp trong Git.

## 20.4 Tránh in secret ra log

Không nên:

```text id="3c2ra6"
printenv
echo $DATABASE_PASSWORD
log toàn bộ environment variables
```

trong production.

Log rất dễ được gom về hệ thống tập trung như ELK, Loki, Cloud Logging. Nếu app in secret ra log thì secret bị lộ rộng hơn.

---

# 21. Troubleshooting nhanh

## 21.1 Pod không start vì thiếu ConfigMap

Triệu chứng:

```text id="rlp0up"
Pod Pending hoặc CreateContainerConfigError
```

Kiểm tra:

```bash id="64xsr2"
kubectl describe pod <pod-name> -n demo
```

Có thể thấy event kiểu:

```text id="2xw2ne"
configmap "app-config" not found
```

Cách xử lý:

```bash id="z636sj"
kubectl get cm -n demo
kubectl apply -f app-config.yaml
```

Đảm bảo ConfigMap và Pod nằm cùng namespace.

## 21.2 Pod không start vì thiếu Secret

Kiểm tra:

```bash id="iw5rgi"
kubectl describe pod <pod-name> -n demo
```

Event có thể là:

```text id="9h82x5"
secret "app-secret" not found
```

Cách xử lý:

```bash id="7o54ge"
kubectl get secret -n demo
kubectl apply -f app-secret.yaml
```

Secret cần tồn tại trước khi Pod phụ thuộc vào nó chạy, trừ khi được khai báo optional.

## 21.3 Sai key trong ConfigMap hoặc Secret

Ví dụ Deployment tham chiếu:

```yaml id="fyg0ni"
key: DATABASE_HOST
```

Nhưng ConfigMap lại có:

```yaml id="ujrpv2"
DB_HOST: "postgres"
```

Pod có thể fail khi startup.

Kiểm tra:

```bash id="l8jhhc"
kubectl describe pod <pod-name> -n demo
kubectl get cm app-config -n demo -o yaml
kubectl get secret app-secret -n demo -o yaml
```

Cách xử lý:

```text id="4yb26n"
Sửa key trong ConfigMap/Secret
hoặc sửa key được reference trong Deployment
```

## 21.4 ConfigMap/Secret nằm sai namespace

ConfigMap/Secret phải nằm cùng namespace với Pod sử dụng nó.

Ví dụ Pod ở namespace `demo`, nhưng ConfigMap lại ở namespace `default`.

Kiểm tra:

```bash id="jlif8u"
kubectl get cm -A | grep app-config
kubectl get secret -A | grep app-secret
```

Cách xử lý:

```text id="2fecmi"
Tạo ConfigMap/Secret trong đúng namespace
hoặc deploy app vào namespace chứa ConfigMap/Secret
```

## 21.5 Đổi ConfigMap nhưng app không nhận giá trị mới

Nếu dùng env, cần restart Pod:

```bash id="n51cj8"
kubectl rollout restart deploy/<deployment-name> -n demo
```

Nếu dùng volume mount, kiểm tra file trong container:

```bash id="y7ofky"
kubectl exec -n demo <pod-name> -- cat /path/to/config/file
```

Nếu file đã đổi nhưng app vẫn dùng config cũ, app có thể chỉ đọc config lúc startup. Cần reload hoặc restart app.

## 21.6 Secret decode ra sai value

Nếu dùng `data`, kiểm tra có encode đúng không:

```bash id="txf16r"
echo -n "demo_password" | base64
```

Lưu ý dùng `echo -n`, vì nếu không có `-n`, newline có thể bị encode vào secret.

Ví dụ:

```bash id="toz6a7"
echo "demo_password" | base64
```

có thể encode cả ký tự xuống dòng.

Nên dùng:

```bash id="qdakwk"
echo -n "demo_password" | base64
```

---

# 22. Bài lab tự làm

## Lab 1: Tạo ConfigMap

Tạo namespace:

```bash id="zbmdik"
kubectl create namespace demo
```

Tạo ConfigMap:

```bash id="mlzh08"
kubectl create configmap app-config \
  -n demo \
  --from-literal=APP_ENV=dev \
  --from-literal=LOG_LEVEL=debug \
  --from-literal=APP_MESSAGE="Hello from ConfigMap"
```

Kiểm tra:

```bash id="bqlsp8"
kubectl get cm -n demo
kubectl describe cm app-config -n demo
```

## Lab 2: Tạo Secret

```bash id="dmwzt1"
kubectl create secret generic app-secret \
  -n demo \
  --from-literal=APP_USERNAME=demo_user \
  --from-literal=APP_PASSWORD=demo_password
```

Kiểm tra:

```bash id="wgxqjt"
kubectl get secret -n demo
kubectl describe secret app-secret -n demo
```

Decode thử:

```bash id="x294pq"
kubectl get secret app-secret -n demo -o jsonpath='{.data.APP_PASSWORD}' | base64 -d
```

## Lab 3: Tạo Deployment dùng ConfigMap và Secret qua env

Tạo file `deployment-env.yaml`:

```yaml id="22ix3f"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-demo
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: env-demo
  template:
    metadata:
      labels:
        app: env-demo
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command: ["sh", "-c", "printenv | grep APP && sleep 3600"]
          envFrom:
            - configMapRef:
                name: app-config
            - secretRef:
                name: app-secret
```

Apply:

```bash id="z2au8f"
kubectl apply -f deployment-env.yaml
```

Xem log:

```bash id="opkl66"
kubectl logs -n demo deploy/env-demo
```

Yêu cầu thấy:

```text id="wldhhb"
APP_ENV=dev
LOG_LEVEL=debug
APP_MESSAGE=Hello from ConfigMap
APP_USERNAME=demo_user
APP_PASSWORD=demo_password
```

## Lab 4: Update ConfigMap và quan sát env không đổi

Update ConfigMap:

```bash id="n67q5k"
kubectl patch cm app-config -n demo --type merge -p '{"data":{"APP_ENV":"prod"}}'
```

Xem log hoặc exec:

```bash id="et0m8u"
kubectl exec -n demo deploy/env-demo -- printenv | grep APP_ENV
```

Có thể vẫn thấy:

```text id="z8dkij"
APP_ENV=dev
```

Restart Deployment:

```bash id="yvkbl4"
kubectl rollout restart deploy/env-demo -n demo
kubectl rollout status deploy/env-demo -n demo
```

Kiểm tra lại:

```bash id="um5ii3"
kubectl exec -n demo deploy/env-demo -- printenv | grep APP_ENV
```

Kỳ vọng:

```text id="xjsf0q"
APP_ENV=prod
```

## Lab 5: Mount ConfigMap thành file

Tạo ConfigMap từ file:

```bash id="2q84j3"
cat > app.properties <<EOF
app.env=dev
log.level=debug
feature.enabled=true
EOF

kubectl create configmap app-file-config \
  -n demo \
  --from-file=app.properties
```

Tạo Pod mount file:

```yaml id="r5z5db"
apiVersion: v1
kind: Pod
metadata:
  name: file-demo
  namespace: demo
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app-config
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: app-file-config
```

Apply:

```bash id="ljzh1z"
kubectl apply -f file-demo.yaml
```

Kiểm tra:

```bash id="csllj2"
kubectl exec -n demo file-demo -- cat /etc/app-config/app.properties
```

## Lab 6: Cố tình làm lỗi thiếu Secret

Sửa Deployment tham chiếu Secret không tồn tại:

```yaml id="5yt64r"
envFrom:
  - secretRef:
      name: missing-secret
```

Apply và kiểm tra:

```bash id="gaq4ks"
kubectl apply -f deployment-env.yaml
kubectl get pods -n demo
kubectl describe pod <pod-name> -n demo
```

Quan sát lỗi liên quan đến Secret không tồn tại.

Sau đó sửa lại `app-secret`.

---

# 23. Checklist cần nắm sau bài này

Sau bài này, cần trả lời được:

```text id="m2ahhu"
ConfigMap dùng để làm gì?
Secret dùng để làm gì?
ConfigMap khác Secret thế nào?
Vì sao không nên hard-code config vào image?
Vì sao không nên lưu password trong ConfigMap?
Pod có thể dùng ConfigMap/Secret bằng những cách nào?
env khác envFrom thế nào?
configMapKeyRef dùng khi nào?
secretKeyRef dùng khi nào?
Mount ConfigMap thành file như thế nào?
Mount Secret thành file như thế nào?
data và stringData trong Secret khác gì?
Base64 có phải encryption không?
Đổi ConfigMap thì Pod có tự nhận không?
Đổi Secret thì Pod có tự nhận không?
Vì sao update env cần restart Pod?
ConfigMap/Secret có cần cùng namespace với Pod không?
Lỗi Secret not found debug thế nào?
Lỗi key không tồn tại debug thế nào?
```

---

# 24. Tóm tắt ngắn gọn

ConfigMap và Secret giúp tách cấu hình runtime ra khỏi container image.

```text id="e4lkyl"
ConfigMap = cấu hình không nhạy cảm
Secret = cấu hình nhạy cảm
```

Pod dùng ConfigMap/Secret qua:

```text id="u7rfej"
Environment variables
Volume mount thành file
```

Cách dùng env từng key:

```yaml id="zwtzqw"
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: APP_ENV
```

Cách dùng Secret từng key:

```yaml id="zatmx0"
env:
  - name: DATABASE_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-secret
        key: DATABASE_PASSWORD
```

Cách dùng toàn bộ:

```yaml id="xbiqcp"
envFrom:
  - configMapRef:
      name: app-config
  - secretRef:
      name: app-secret
```

Cách mount thành file:

```yaml id="r6081s"
volumes:
  - name: config-volume
    configMap:
      name: app-config
  - name: secret-volume
    secret:
      secretName: app-secret
```

Những điểm cần nhớ:

```text id="eybbcd"
Không lưu password/token trong ConfigMap.
Không commit Secret thật vào Git.
Base64 không phải encryption.
ConfigMap/Secret phải cùng namespace với Pod.
Nếu dùng env, update ConfigMap/Secret cần restart Pod.
Nếu dùng volume mount, file có thể update nhưng app có thể vẫn cần reload/restart.
```

Hiểu chắc ConfigMap và Secret là nền tảng để học tiếp các phần như Volume, PVC, Ingress, Helm, CI/CD và secret management trong production.
