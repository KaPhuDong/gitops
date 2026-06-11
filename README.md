# GitOps - Observability - Canary

Repo này triển khai hệ thống theo mô hình GitOps với Argo CD App-of-Apps, Prometheus/Grafana để quan sát hệ thống, và Argo Rollouts để phát hành Canary tự động cho API.

## Cấu Trúc Repo

```text
gitops/
├── argocd/
│   ├── root.yaml
│   └── apps/
│       ├── web.yaml
│       ├── backend.yaml
│       ├── frontend.yaml
│       ├── kube-prometheus-stack.yaml
│       ├── argo-rollouts.yaml
│       └── api.yaml
├── app/
│   ├── app.py
│   └── Dockerfile
├── k8s/
│   ├── namespace.yaml
│   ├── web.yaml
│   ├── backend/
│   │   └── app.yaml
│   └── frontend/
│       └── app.yaml
└── k8s-api/
    ├── api.yaml
    ├── servicemonitor.yaml
    ├── analysis.yaml
    └── alerts.yaml
```

## Những Giá Trị Cần Thay Trước Khi Demo

Trong các file Argo CD Application:

- `https://github.com/KaPhuDong/gitops.git`: thay bằng URL repo Git thật nếu repo đổi.

Trong `argocd/apps/kube-prometheus-stack.yaml`:

- `TODO_YOUR_GMAIL@gmail.com`: thay bằng Gmail dùng để gửi alert.
- `TODO_GMAIL_APP_PASSWORD`: thay bằng Gmail App Password thật.
- `TODO_RECEIVER_EMAIL@example.com`: thay bằng email nhận alert.

Trong `k8s-api/api.yaml`:

- `image: w9-api:1`: giữ nếu dùng Minikube local image. Nếu chạy trên cluster khác, thay bằng image thật trên registry.

## Deploy Bằng GitOps

Apply root Application một lần:

```bash
kubectl apply -f argocd/root.yaml
```

Root app sẽ đọc `argocd/apps/` và tạo các app con:

- `kube-prometheus-stack`
- `argo-rollouts`
- `web`
- `be`
- `fe`
- `api`

Kiểm tra:

```bash
kubectl -n argocd get applications
kubectl -n monitoring get pods
kubectl -n argo-rollouts get pods
kubectl -n demo get pods
```

## Build API Image

Nếu dùng Minikube:

```bash
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9
```

API Flask có các endpoint:

- `/`: trả response thành công hoặc lỗi giả lập.
- `/healthz`: readiness probackend.
- `/metrics`: Prometheus scrape metric.

Biến môi trường:

- `VERSION`: version API.
- `ERROR_RATE`: tỷ lệ lỗi giả lập, ví dụ `0`, `0.2`, `0.5`.

## Canary Tự Động Cho API

File chính:

```text
k8s-api/api.yaml
```

Rollout `api` dùng strategy Canary:

- 25% traffic.
- Chạy analysis tự động.
- 50% traffic.
- Chạy analysis tự động.
- Nếu metric đạt thì lên 100%.
- Nếu metric xấu thì tự abort.

Điểm quan trọng: file này không còn dùng `pause: {}` thủ công. Canary được chấm điểm bằng `AnalysisTemplate`.

## SLO Và Analysis

File:

```text
k8s-api/analysis.yaml
```

SLO:

```text
Success Rate >= 99%
```

PromQL:

```promql
(
  sum(rate(flask_http_request_total{app="api",status!~"5.."}[1m]))
  /
  sum(rate(flask_http_request_total{app="api"}[1m]))
) or vector(1)
```

Điều kiện:

```yaml
interval: 30s
successCondition: result[0] >= 0.99
failureLimit: 2
```

Nghĩa là nếu success rate dưới 99% trong 2 lần kiểm tra, Rollout sẽ abort.

## Prometheus Scrape API

File:

```text
k8s-api/servicemonitor.yaml
```

Prometheus scrape:

- Service label: `app=api`
- Port: `http`
- Path: `/metrics`
- Interval: `15s`

## Alert SLO

File:

```text
k8s-api/alerts.yaml
```

Alert `ApiHighErrorRate` fire khi success rate API dưới 99% trong 30 giây.

Alertmanager SMTP được cấu hình trong:

```text
argocd/apps/kube-prometheus-stack.yaml
```

## Test Auto-Abort

Sửa `k8s-api/api.yaml` để tạo bản lỗi:

```yaml
env:
  - name: ERROR_RATE
    value: '0.5'
  - name: VERSION
    value: 'v2'
```

Commit và push:

```bash
git add k8s-api/api.yaml
git commit -m "Test api canary auto abort"
git push
```

Theo dõi:

```bash
kubectl argo rollouts get rollout api -n demo --watch
```

Kỳ vọng:

- Canary bắt đầu.
- Prometheus ghi nhận lỗi `5xx`.
- Analysis fail khi success rate `< 99%`.
- Rollout tự abort về bản ổn định trước đó.
- Alert `ApiHighErrorRate` fire và gửi email.

## Rollback Đúng Theo GitOps

Rollback bằng Git:

```bash
git revert HEAD --no-edit
git push
```

Không dùng:

```bash
kubectl rollout undo
```

Vì nếu rollback trực tiếp trong cluster, Git vẫn giữ trạng thái lỗi và Argo CD sẽ self-heal cluster quay lại theo Git.

## Checklist Đạt Bài

- Argo CD root và app con `Synced/Healthy`.
- Prometheus/Grafana chạy trong namespace `monitoring`.
- Argo Rollouts controller chạy trong namespace `argo-rollouts`.
- API là `Rollout`.
- Prometheus target API ở trạng thái `UP`.
- Query `flask_http_request_total{namespace="demo"}` có dữ liệu.
- Canary bản lỗi tự abort.
- Alert SLO fire.
- Email alert gửi thành công.
- Rollback bằng `git revert` hoàn tất dưới 5 phút.
