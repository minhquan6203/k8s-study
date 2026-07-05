# K8s Study Blog

Repo blog ghi lại hành trình học Kubernetes theo lộ trình từ cơ bản đến triển khai production.

## Mục tiêu

- Học có hệ thống, mỗi chủ đề 1 bài + ví dụ YAML + bài lab.
- Tập trung vào kiến thức dùng được ngay khi đi làm.
- Ưu tiên nội dung thực chiến: debug, vận hành, CI/CD.

## Stack tài liệu

- MkDocs
- Material for MkDocs

## Cấu trúc

- `docs/`: toàn bộ bài viết
- `mkdocs.yml`: điều hướng và cấu hình site
- `requirements.txt`: dependencies để chạy local

## Chạy local

### 1) Tạo môi trường và cài thư viện

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2) Chạy docs server

```bash
mkdocs serve
```

Mở trình duyệt tại địa chỉ: http://127.0.0.1:8000

## Build static site

```bash
mkdocs build
```

Output nằm ở thư mục `site/`.

## Lộ trình bài viết

1. Pod, Deployment, ReplicaSet
2. Service, DNS, Namespace
3. ConfigMap, Secret
4. Volume, PVC, StorageClass
5. Resource requests/limits, quota
6. StatefulSet cho Kafka/MinIO/DB
7. RBAC, ServiceAccount
8. Ingress, NodePort, LoadBalancer
9. Health check, rollout, rollback
10. Helm
11. Monitoring/logging
12. Spark on Kubernetes
13. Kafka/MinIO/Airflow trên Kubernetes
14. CI/CD deploy lên Kubernetes

## Cách viết mỗi bài

- Why: Khi nào cần dùng?
- Core concepts: Khái niệm cốt lõi.
- Manifest: YAML ví dụ.
- Lab: Bài thực hành nhanh.
- Pitfalls: Lỗi thường gặp.
- Checklist: Những gì cần nhớ.
