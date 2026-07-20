# Ingress, NodePort và LoadBalancer trong Kubernetes: Cách expose ứng dụng ra bên ngoài Cluster

Ở các bài trước, chúng ta đã hiểu:

* **Pod, Deployment, ReplicaSet**: cách Kubernetes tạo và quản lý container.
* **Service, DNS, Namespace**: cách các ứng dụng giao tiếp với nhau bên trong cluster.
* **ConfigMap, Secret**: quản lý cấu hình và thông tin nhạy cảm.
* **Volume, PVC, StorageClass**: lưu trữ dữ liệu bền vững.
* **Resource Requests, Limits, Quota**: kiểm soát tài nguyên.
* **StatefulSet, Kafka, MinIO, Database**: chạy workload stateful.
* **RBAC, ServiceAccount**: kiểm soát quyền truy cập.

Tuy nhiên, chúng ta mới chỉ nói về việc:

```text
Pod gọi Pod
Service gọi Service
Application giao tiếp bên trong Kubernetes Cluster
```

Vậy câu hỏi tiếp theo là:

> Làm sao người dùng bên ngoài Internet truy cập được ứng dụng chạy trong Kubernetes?

Ví dụ:

Bạn có một hệ thống:

```text
Frontend React
Backend Spring Boot
API Gateway
Payment Service
User Service
```

đang chạy trong Kubernetes:

```
             Kubernetes Cluster

          frontend-pod
              |
              |
          backend-service
              |
              |
          backend-pod


Internet User
      ?
      |
      |
      v

Làm sao truy cập vào frontend?
```

Pod không thể được truy cập trực tiếp vì:

* Pod IP chỉ tồn tại trong cluster.
* Pod có thể restart và đổi IP.
* Không thể expose hàng trăm Pod ra Internet.

Kubernetes cung cấp nhiều cách để expose application:

```text
Pod
 |
 |
 v
Service
 |
 +----------------+
 |                |
 v                v
NodePort       LoadBalancer
                    |
                    v
                Internet


Hoặc:

Internet
    |
    v
 Ingress
    |
    v
 Service
    |
    v
 Pods
```

Ba khái niệm quan trọng:

```text
NodePort
    =
Expose Service qua port trên Node


LoadBalancer
    =
Cloud Load Balancer đứng trước Service


Ingress
    =
HTTP Reverse Proxy định tuyến request tới nhiều Service
```

Bài này sẽ giải thích:

1. Vì sao cần expose service
2. NodePort hoạt động như thế nào
3. LoadBalancer hoạt động như thế nào
4. Ingress là gì
5. Ingress Controller
6. Routing bằng hostname và path
7. TLS/HTTPS trong Ingress
8. So sánh Ingress vs NodePort vs LoadBalancer
9. Kiến trúc production thực tế
10. Troubleshooting

---

# 1. Vấn đề: Service mặc định chỉ dùng được trong Cluster

Ở bài Service trước, chúng ta có:

```
frontend-pod

      |
      |
      v

backend-service

      |
      |
      v

backend-pods
```

Service mặc định:

```yaml
type: ClusterIP
```

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

Service tạo ra:

```
backend

ClusterIP:

10.96.20.10
```

Nhưng IP này chỉ tồn tại trong cluster.

Nếu user bên ngoài gọi:

```
http://10.96.20.10
```

sẽ không được.

Vì:

```
Internet
    |
    X
    |
ClusterIP
```

ClusterIP dành cho:

```
Pod -> Service
Service -> Service
```

không dành cho:

```
Internet -> Service
```

---

# 2. Các cách expose ứng dụng ra ngoài

Kubernetes có 3 cách phổ biến:

```
1. NodePort

2. LoadBalancer

3. Ingress
```

---

Kiến trúc tổng quát:

```
                    Internet

                       |
                       |

              External Access Layer


        +--------------+---------------+

        |              |               |

     NodePort    LoadBalancer      Ingress


        |              |               |

        +--------------+---------------+

                       |

                    Service


                       |

                    Pods
```

---

# 3. NodePort Service

## 3.1 NodePort là gì?

NodePort là cách đơn giản nhất để expose Service ra bên ngoài.

Khi tạo NodePort:

Kubernetes sẽ:

1. Tạo ClusterIP Service.
2. Mở thêm một port trên tất cả Node.
3. Forward traffic vào Service.

Ví dụ:

Service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: frontend

spec:

  type: NodePort

  selector:
    app: frontend

  ports:

  - port: 80

    targetPort: 3000

    nodePort: 30080
```

---

Sau khi tạo:

```
Service:

frontend

ClusterIP:
10.96.10.20


NodePort:

30080
```

---

Cluster có node:

```
Node 1:
192.168.1.10


Node 2:
192.168.1.11


Node 3:
192.168.1.12
```

Người dùng truy cập:

```
http://192.168.1.10:30080
```

hoặc:

```
http://192.168.1.11:30080
```

hoặc:

```
http://192.168.1.12:30080
```

Đều được.

---

Luồng request:

```
Browser

 |
 |
 v

Node IP:30080

 |
 |
 v

NodePort Service

 |
 |
 v

ClusterIP

 |
 |
 v

Pod
```

---

# 4. NodePort hoạt động bên trong như thế nào?

Giả sử có:

```
frontend-pod-1
10.244.1.10

frontend-pod-2
10.244.2.20
```

Service:

```
frontend-service
```

NodePort:

```
30080
```

Traffic:

```
Client

 |
 v

192.168.1.10:30080

 |
 v

kube-proxy rule

 |
 +------------+
 |            |
 v            v

pod-1       pod-2
```

kube-proxy tạo network rule để:

```
NodePort
    |
    v
Service
    |
    v
Pod endpoint
```

---

# 5. Port trong NodePort

NodePort có 3 port:

```text
nodePort
port
targetPort
```

Ví dụ:

```yaml
ports:

- port: 80

  targetPort: 8080

  nodePort: 30080
```

Ý nghĩa:

## nodePort

Port mở trên Node.

User truy cập:

```
NodeIP:30080
```

---

## port

Port của Service.

```
Service:80
```

---

## targetPort

Port trong container.

```
Pod:8080
```

---

Luồng:

```
Client

192.168.1.10:30080

        |

        v

Service port 80

        |

        v

Container port 8080
```

---

# 6. Nhược điểm của NodePort

NodePort rất đơn giản nhưng production thường ít dùng trực tiếp.

## 6.1 Port khó nhớ

Ví dụ:

```
http://company.com:30080
```

không đẹp.

---

## 6.2 Chỉ expose bằng IP

Không có:

```
api.company.com
```

mà phải:

```
192.168.1.10:30080
```

---

## 6.3 Quản lý nhiều service khó

Ví dụ:

100 service:

```
frontend :30001

backend :30002

payment :30003

user :30004

...
```

Rất khó quản lý.

---

Vì vậy NodePort thường dùng:

```
Development

Testing

Lab

On-premise nhỏ
```

---

# 7. LoadBalancer Service

## 7.1 LoadBalancer là gì?

LoadBalancer là Service type dùng để tạo một external load balancer bên ngoài cluster.

Ví dụ cloud:

```
AWS

GCP

Azure
```

Kubernetes yêu cầu cloud provider tạo:

```
External Load Balancer
```

---

Manifest:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: frontend

spec:

  type: LoadBalancer

  selector:
    app: frontend

  ports:

  - port: 80

    targetPort: 3000
```

---

Sau khi tạo:

```
kubectl get svc
```

Ví dụ:

```
NAME

frontend


TYPE

LoadBalancer


EXTERNAL-IP

34.120.20.10
```

---

User truy cập:

```
http://34.120.20.10
```

---

Luồng:

```
Internet

    |

    v

Cloud LoadBalancer

    |

    v

Kubernetes Service

    |

    v

Pods
```

---

# 8. LoadBalancer trong AWS/GCP/Azure

Ví dụ AWS:

```
Internet

    |

    v

AWS ELB

    |

    v

Kubernetes Service

    |

    v

Pods
```

Cloud provider chịu trách nhiệm:

* Public IP
* Health check
* Routing
* Availability

---

# 9. Nhược điểm của LoadBalancer

Giả sử:

Bạn có:

```
frontend-service

backend-service

payment-service

user-service
```

Nếu mỗi service tạo LoadBalancer:

```
frontend
    |
AWS ELB


backend
    |
AWS ELB


payment
    |
AWS ELB
```

Chi phí rất lớn.

---

Production thường không làm vậy.

Thay vào đó:

```
Một LoadBalancer

       |

       v

Ingress Controller

       |

       +------ frontend
       |
       +------ backend
       |
       +------ api
```

Đây là lý do Ingress ra đời.

---

# 10. Ingress là gì?

## 10.1 Khái niệm

Ingress là Kubernetes object dùng để quản lý HTTP/HTTPS routing.

Nó cho phép:

```
Một IP public

        |

        v

Nhiều Service
```

---

Ví dụ:

Bạn có:

```
frontend-service

backend-service

payment-service
```

Thay vì:

```
frontend LoadBalancer

backend LoadBalancer

payment LoadBalancer
```

Ta dùng:

```
             Internet

                 |

                 v

        Ingress Controller

                 |

        +--------+---------+

        |        |         |

        v        v         v

 frontend   backend   payment

 service    service   service
```

---

# 11. Ingress Controller là gì?

Rất nhiều người nhầm:

```
Ingress = chạy được ngay
```

Không đúng.

Ingress chỉ là:

```
Configuration object
```

Nó mô tả rule.

Muốn chạy được cần:

```
Ingress Controller
```

---

Ví dụ Ingress Controller:

```
NGINX Ingress Controller

Traefik

HAProxy

AWS ALB Controller

Istio Gateway
```

---

Luồng:

```
Ingress YAML

      |

      v

Ingress Controller

      |

      v

Service
```

---

# 12. Ví dụ Ingress cơ bản

Giả sử:

Frontend:

```
frontend-service
```

Backend:

```
backend-service
```

Ta muốn:

```
example.com

        |

        +---- /      -> frontend

        |
        +---- /api   -> backend
```

---

Ingress:

```yaml
apiVersion: networking.k8s.io/v1

kind: Ingress


metadata:

  name: app-ingress


spec:


  rules:


  - host:

      example.com


    http:


      paths:


      - path: /

        pathType: Prefix

        backend:


          service:

            name: frontend-service

            port:

              number: 80



      - path: /api

        pathType: Prefix


        backend:


          service:

            name: backend-service


            port:

              number: 80
```

---

Kết quả:

Request:

```
https://example.com/
```

đi:

```
frontend-service
```

---

Request:

```
https://example.com/api/users
```

đi:

```
backend-service
```

---

# 13. Path Based Routing

Một domain nhiều service.

Ví dụ:

```
company.com

/
   frontend


/api
   backend


/payment
   payment-service
```

---

Ingress:

```
Request path

        |

        v

Ingress Controller


        |

        +---- /
        |
        +---- /api
        |
        +---- /payment
```

---

Đây là cách rất phổ biến cho microservice.

---

# 14. Host Based Routing

Ngoài path, có thể route theo domain.

Ví dụ:

```
frontend.company.com

        |
        v

frontend-service


api.company.com

        |
        v

backend-service
```

---

Ingress:

```yaml
rules:

- host:
    api.company.com

  http:

    paths:

    - path: /

      backend:

        service:

          name: backend


- host:

    frontend.company.com

  http:

    paths:

    - path: /

      backend:

        service:

          name: frontend
```

---

# 15. HTTPS và TLS trong Ingress

Production gần như luôn dùng HTTPS.

Luồng:

```
User

https://api.company.com

        |

        v

Ingress Controller

        |

        v

Service

        |

        v

Pod
```

---

Ingress quản lý certificate:

Ví dụ:

```yaml
tls:

- hosts:

  - api.company.com


  secretName:

    api-tls
```

Certificate lưu trong:

```
Secret
```

---

Secret:

```
api-tls

|
+-- tls.crt

+-- tls.key
```

---

# 16. Ingress Controller expose như thế nào?

Bản thân Ingress Controller cũng cần được expose.

Ví dụ:

```
Internet

   |

   v

LoadBalancer Service

   |

   v

Ingress Controller Pod

   |

   v

Application Service
```

---

Thực tế:

```
Một LoadBalancer

       |

       v

NGINX Ingress Controller

       |

       +---- app1
       |
       +---- app2
       |
       +---- app3
```

---

# 17. So sánh NodePort, LoadBalancer, Ingress

|                 | NodePort | LoadBalancer | Ingress      |
| --------------- | -------- | ------------ | ------------ |
| Layer           | L4       | L4           | L7           |
| HTTP routing    | Không    | Không        | Có           |
| Domain          | Không    | Có thể       | Có           |
| SSL termination | Không    | Có thể       | Có           |
| Nhiều service   | Khó      | Tốn tiền     | Rất tốt      |
| Production      | Ít dùng  | Dùng         | Rất phổ biến |

---

# 18. Layer 4 và Layer 7 khác nhau thế nào?

Đây là khái niệm quan trọng.

## Layer 4

TCP/UDP.

Ví dụ:

```
IP + Port
```

LoadBalancer hoạt động ở đây.

Ví dụ:

```
34.20.10.5:443
```

---

## Layer 7

HTTP.

Ingress hiểu:

```
Host

Path

Header

Cookie
```

Ví dụ:

```
GET /api/user

Host:
api.company.com
```

Ingress biết:

```
api.company.com
+
/api

=> backend service
```

---

# 19. Kiến trúc Production phổ biến

Một hệ thống Kubernetes thực tế:

```
                 Internet

                    |

                    v


          Cloud LoadBalancer


                    |

                    v


          Ingress Controller


                    |

       +------------+-------------+

       |            |             |

       v            v             v


 frontend      backend       payment

 Service       Service      Service


       |            |             |


       v            v             v


    Pods         Pods          Pods
```

---

Đây là kiến trúc phổ biến:

```
1 LoadBalancer

1 Ingress Controller

Nhiều Service

Nhiều Deployment
```

---

# 20. Khi nào dùng cái gì?

## Dùng NodePort khi:

```
- Test nhanh
- Local cluster
- Minikube
- Kind
- Lab
```

---

## Dùng LoadBalancer khi:

```
- Cloud Kubernetes
- Muốn expose service trực tiếp
- Service ít
```

---

## Dùng Ingress khi:

```
- Production
- Nhiều HTTP service
- Có domain
- Có HTTPS
- Microservice architecture
```

---

# 21. Troubleshooting

## 21.1 Ingress không truy cập được

Kiểm tra:

```bash
kubectl get ingress
```

---

Kiểm tra controller:

```bash
kubectl get pods -n ingress-nginx
```

---

Kiểm tra service:

```bash
kubectl get svc -n ingress-nginx
```

---

# 21.2 Ingress trả 404

Nguyên nhân:

```
Sai host

Sai path

Sai service name

Sai port
```

---

Kiểm tra:

```bash
kubectl describe ingress
```

---

# 21.3 Service không có endpoint

Kiểm tra:

```bash
kubectl get endpoints
```

Nếu:

```
<none>
```

thì:

```
Ingress đúng
nhưng Service không có Pod phía sau
```

---

# 21.4 LoadBalancer Pending

Ví dụ:

```
EXTERNAL-IP

<pending>
```

Nguyên nhân:

```
Cluster không có cloud provider

Không có MetalLB

Không có controller
```

---

# 21.5 NodePort không truy cập được

Kiểm tra:

```bash
kubectl get svc
```

Kiểm tra:

```
Firewall

Security Group

Node IP

Endpoint
```

---

# 22. Bài lab tự làm

## Lab 1: NodePort

Tạo:

```
nginx Deployment

Service NodePort
```

Truy cập:

```
NodeIP:30080
```

---

## Lab 2: LoadBalancer

Tạo:

```
Service type LoadBalancer
```

Quan sát:

```
EXTERNAL-IP
```

---

## Lab 3: Cài NGINX Ingress Controller

Cài:

```
nginx ingress controller
```

---

## Lab 4: Routing path

Tạo:

```
/
frontend-service


/api
backend-service
```

Test:

```
curl example.com

curl example.com/api
```

---

## Lab 5: TLS

Tạo:

```
TLS Secret

Ingress HTTPS
```

Test:

```
https://example.com
```

---

# 23. Checklist cần nắm sau bài này

Sau bài này cần trả lời được:

```
Vì sao ClusterIP không truy cập từ Internet?

NodePort hoạt động thế nào?

LoadBalancer khác NodePort ra sao?

Ingress là gì?

Ingress Controller làm gì?

Ingress có tự chạy không?

Path routing là gì?

Host routing là gì?

TLS trong Ingress hoạt động thế nào?

Layer 4 và Layer 7 khác nhau thế nào?

Production Kubernetes thường expose app bằng cách nào?

Khi nào dùng NodePort?

Khi nào dùng LoadBalancer?

Khi nào dùng Ingress?
```

---

# 24. Tóm tắt ngắn gọn

Cách expose application trong Kubernetes:

```
Pod
 |
 |
 v

Service

 |
 +----------------+
 |                |
 v                v

NodePort       LoadBalancer


hoặc


Internet

   |

   v

Ingress Controller

   |

   v

Service

   |

   v

Pods
```

Nhớ nhanh:

```
NodePort
=
Mở port trên Node


LoadBalancer
=
Cloud cấp IP public


Ingress
=
HTTP reverse proxy route nhiều service
```

Trong môi trường production hiện nay:

```
Internet

    |

LoadBalancer

    |

Ingress Controller

    |

Services

    |

Deployments

    |

Pods
```

Đây là kiến trúc Kubernetes phổ biến nhất khi triển khai hệ thống microservice thực tế.
