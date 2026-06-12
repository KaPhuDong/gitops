# Flow Dự Án

Tài liệu này mô tả luồng hoạt động của dự án Ship Smartly: từ Git commit đến Kubernetes deployment, monitoring, alerting và test bằng frontend.

## Kiến Trúc Tổng Quan

```text
Developer
  |
  | git commit / git push
  v
GitHub repository
  |
  | Argo CD poll Git
  v
Argo CD root app
  |
  | app-of-apps
  v
Child Applications
  |
  +-- frontend -> k8s/frontend.yaml
  +-- api -> k8s-api/
  +-- backend -> k8s/backend.yaml
  +-- web -> k8s/web.yaml
  +-- kube-prometheus-stack -> Prometheus/Grafana/Alertmanager
  +-- argo-rollouts -> Rollouts controller
```

Git là nguồn sự thật duy nhất. Trạng thái Kubernetes phải được thay đổi bằng cách sửa file, commit, push, rồi để Argo CD reconcile cluster.

## Cấu Trúc Repo

```text
argocd/
  root.yaml
  apps/
    api.yaml
    frontend.yaml
    backend.yaml
    web.yaml
    kube-prometheus-stack.yaml
    argo-rollouts.yaml

k8s/
  namespace.yaml
  frontend.yaml
  backend.yaml
  web.yaml

k8s-api/
  api.yaml
  servicemonitor.yaml
  analysis.yaml
  alerts.yaml

app/
  app.py
  Dockerfile
```

## Flow GitOps

1. Developer sửa manifest trong repo.
2. Commit và push thay đổi lên GitHub.
3. Argo CD phát hiện Git revision mới.
4. App `root` sync các child Application trong `argocd/apps/`.
5. Mỗi child Application sync resource của nó.
6. Nếu ai sửa tay trong cluster, Argo CD phát hiện drift và self-heal về trạng thái trong Git.

Root Application trỏ tới:

```text
argocd/apps/
```

Mỗi app có phạm vi quản lý hẹp:

```text
frontend -> k8s/frontend.yaml
api      -> k8s-api/
backend  -> k8s/backend.yaml
web      -> k8s/web.yaml
```

Cách này tránh nhiều app cùng quản lý trùng một resource.

## Flow Frontend

Frontend được định nghĩa tại:

```text
k8s/frontend.yaml
```

Trong đó có:

- `ConfigMap/fe-page`: chứa HTML, CSS và JavaScript của UI.
- `ConfigMap/fe-nginx-config`: cấu hình nginx reverse proxy.
- `Service/fe-svc`: expose frontend trong cluster ở port `8081`.
- `Deployment/fe`: chạy `nginx:alpine`.

Truy cập local:

```bash
kubectl -n demo port-forward svc/fe-svc 8081:8081
```

Mở:

```text
http://localhost:8081
```

Browser gọi:

```text
/api/
```

nginx proxy request đó tới:

```text
http://api.demo.svc.cluster.local:8080/
```

Nhờ vậy browser chỉ gọi cùng origin với frontend, không bị vướng CORS.

## Flow API

Source API:

```text
app/app.py
```

Kubernetes rollout:

```text
k8s-api/api.yaml
```

API endpoints:

```text
GET /         -> trả JSON success hoặc injected error
GET /healthz  -> readiness probe
GET /metrics  -> metric cho Prometheus
```

Biến môi trường điều khiển runtime:

```yaml
env:
  - name: ERROR_RATE
    value: '0'
  - name: VERSION
    value: 'v1'
```

Ý nghĩa:

- `ERROR_RATE=0`: API trả success.
- `ERROR_RATE=0.5`: khoảng 50% request trả HTTP 500.
- `VERSION`: giúp chứng minh version đang chạy.

## Flow Observability

Prometheus stack được cài bởi:

```text
argocd/apps/kube-prometheus-stack.yaml
```

API được Prometheus phát hiện qua:

```text
k8s-api/servicemonitor.yaml
```

Đường scrape:

```text
Prometheus -> ServiceMonitor/api-monitor -> Service/api:8080 -> Pod/api:/metrics
```

Metric quan trọng:

```promql
flask_http_request_duration_seconds_count{namespace="demo"}
```

Tổng số request:

```promql
sum(flask_http_request_duration_seconds_count{namespace="demo"})
```

Success rate:

```promql
sum(rate(flask_http_request_duration_seconds_count{status!~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
```

Error rate:

```promql
sum(rate(flask_http_request_duration_seconds_count{status=~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
```

## Flow Test Bằng Frontend

Frontend thay thế các vòng lặp curl thủ công để tạo traffic.

Flow request đơn:

```text
User bấm "Call API"
  -> browser fetch /api/
  -> nginx proxy tới api.demo.svc.cluster.local:8080
  -> Flask API trả JSON
  -> UI hiển thị response
  -> Prometheus scrape metric ở lần scrape tiếp theo
```

Flow burst:

```text
User bấm "Burst 100 Calls"
  -> browser gửi 100 request theo batch
  -> API sinh request metric
  -> Prometheus scrape /metrics
  -> AnalysisTemplate và PrometheusRule dùng dữ liệu mới
```

Nút burst chỉ tạo traffic. Việc có alert hay không phụ thuộc vào API hiện tại có trả đủ lỗi 5xx hay không.

## Flow Canary Và Analysis

Argo Rollouts được cài bởi:

```text
argocd/apps/argo-rollouts.yaml
```

API rollout nằm ở:

```text
k8s-api/api.yaml
```

Chiến lược rollout:

```text
25% traffic -> pause 1m -> 50% traffic -> pause 30s -> 100%
```

Analysis template:

```text
k8s-api/analysis.yaml
```

Nó query Prometheus mỗi 10 giây và yêu cầu:

```text
success rate >= 0.95
```

Nếu success rate thấp hơn 0.95 quá số lần cho phép, rollout sẽ abort.

Flow bản tốt:

```text
ERROR_RATE=0
VERSION=v2-good
  -> FE Burst tạo request
  -> success rate gần 1
  -> analysis pass
  -> rollout promote lên 100%
```

Flow bản lỗi:

```text
ERROR_RATE=0.5
VERSION=v2-bad
  -> FE Burst tạo request
  -> 5xx rate tăng
  -> success rate tụt dưới 0.95
  -> analysis fail
  -> rollout abort
  -> stable ReplicaSet vẫn phục vụ traffic
```

## Flow Alert

Alert rule:

```text
k8s-api/alerts.yaml
```

Điều kiện alert:

```text
error rate > 5% trong 30 giây
```

Flow:

```text
API trả 5xx
  -> prometheus-flask-exporter expose metric
  -> Prometheus scrape metric
  -> PrometheusRule đánh giá error rate
  -> alert chuyển Pending
  -> sau 30s chuyển Firing
  -> Alertmanager nhận alert
  -> Alertmanager gửi email
```

Email receiver:

```text
kaphudong04@gmail.com
```

Lưu ý:

- Gửi email cần Gmail App Password thật.
- Không commit password thật lên repo public.
- Nếu vẫn để `TODO_GMAIL_APP_PASSWORD`, alert có thể firing nhưng email sẽ không gửi thành công.

## Flow Rollback

Rollback phải đi qua Git:

```bash
git revert HEAD --no-edit
git push
```

Flow:

```text
git revert
  -> tạo commit mới khôi phục desired state cũ
  -> Argo CD sync manifest đã revert
  -> rollout trở về version khỏe
```

Không dùng cách này làm rollback chính:

```bash
kubectl rollout undo
```

Lý do: `kubectl rollout undo` chỉ sửa cluster. Git vẫn giữ desired state lỗi, nên Argo CD có thể sync bản lỗi quay lại.

## Flow CI Validate

GitHub Actions validate manifest bằng:

```bash
kubeconform -strict -ignore-missing-schemas -summary argocd/ k8s/ k8s-api/
```

Mục tiêu:

- Bắt lỗi YAML.
- Bắt lỗi schema Kubernetes cơ bản.
- Chặn merge khi manifest lỗi.

Ví dụ lỗi đã từng gặp:

```yaml
image: image: w9-api:1
```

Dạng đúng:

```yaml
image: w9-api:1
```

## Kịch Bản Demo Đề Xuất

1. Show Argo CD apps đều Synced/Healthy.
2. Show FE tại `http://localhost:8081`.
3. Bấm `Call API`, show JSON response.
4. Bấm `Burst 100 Calls`.
5. Show Prometheus metric tăng.
6. Deploy good canary và show promote.
7. Deploy bad canary với `ERROR_RATE=0.5`.
8. Bấm `Burst 100 Calls` khi canary đang chạy.
9. Show analysis failure và rollout abort.
10. Show Prometheus alert firing.
11. Show Alertmanager/email.
12. Chạy `git revert` và show rollback.
