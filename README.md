# Challenge: Ship Smartly

Repo này triển khai bài Challenge "Ship Smartly" theo 3 phần chính:

- **GitOps**: mọi thay đổi đi qua Git, Argo CD tự sync và self-heal.
- **Observability**: Prometheus/Grafana thu thập metric, có SLO và alert.
- **Progressive Delivery**: API dùng Argo Rollouts Canary, bản lỗi tự abort dựa trên Prometheus metric.

Mục tiêu đạt bài:

- Argo CD quản lý toàn bộ manifest theo mô hình App-of-Apps.
- API có endpoint `/metrics` để Prometheus scrape.
- Có SLO success rate và alert gửi email cá nhân.
- Canary không còn pause thủ công vô hạn; hệ thống tự chấm điểm bằng `AnalysisTemplate`.
- Bản tốt được promote lên 100%, bản lỗi tự abort về bản cũ.
- Rollback bằng `git revert` trong dưới 5 phút.

## Cấu Trúc Repo

Cấu trúc hiện tại đã đưa các manifest app thường ra trực tiếp trong thư mục `k8s/` và đổi tên thành file riêng:

```text
gitops/
|-- argocd/
|   |-- root.yaml
|   `-- apps/
|       |-- web.yaml
|       |-- backend.yaml
|       |-- frontend.yaml
|       |-- kube-prometheus-stack.yaml
|       |-- argo-rollouts.yaml
|       `-- api.yaml
|-- app/
|   |-- app.py
|   `-- Dockerfile
|-- k8s/
|   |-- namespace.yaml
|   |-- web.yaml
|   |-- backend.yaml
|   `-- frontend.yaml
`-- k8s-api/
    |-- api.yaml
    |-- servicemonitor.yaml
    |-- analysis.yaml
    `-- alerts.yaml
```

## App-of-Apps

Root Application:

```text
argocd/root.yaml
```

Root trỏ tới:

```text
argocd/apps/
```

Các app con:

- `kube-prometheus-stack`: cài Prometheus, Grafana, Alertmanager.
- `argo-rollouts`: cài Argo Rollouts controller.
- `web`: deploy `k8s/web.yaml`.
- `be`: deploy `k8s/backend.yaml`.
- `frontend`: deploy `k8s/frontend.yaml`.
- `api`: deploy toàn bộ `k8s-api/`.

Vì `web`, `backend`, `frontend` hiện là các file nằm chung trong `k8s/`, các Argo CD Application tương ứng dùng:

```yaml
source:
  path: k8s
  directory:
    include: <ten-file>.yaml
```

Cách này giúp mỗi app chỉ sync đúng file của nó, tránh deploy trùng tài nguyên.

## Những Giá Trị Cần Thay

Kiểm tra và thay các giá trị thật trước khi demo:

- `argocd/root.yaml` và `argocd/apps/*.yaml`: thay `repoURL` nếu repo GitHub thay đổi.
- `argocd/apps/kube-prometheus-stack.yaml`: thay SMTP Gmail, Gmail App Password và email nhận alert.
- `k8s-api/api.yaml`: thay `image: w9-api:1` bằng image registry thật nếu không dùng Minikube local image.

Lưu ý bảo mật: không nên commit Gmail App Password thật vào repo public. Nếu đã push secret thật, nên revoke app password đó và tạo app password mới.

## Deploy Bằng GitOps

Apply root Application một lần:

```bash
kubectl apply -f argocd/root.yaml
```

Kiểm tra Argo CD:

```bash
kubectl -n argocd get applications
kubectl -n argocd get application root
kubectl -n argocd get application api
```

Kỳ vọng:

- `root` ở trạng thái `Synced/Healthy`.
- Các app con ở trạng thái `Synced/Healthy`.
- Không có drift giữa Git và cluster.

## API Có Metrics

Source API nằm ngoài thư mục manifest Kubernetes, tại:

```text
app/app.py
app/Dockerfile
```

API Flask có các endpoint:

- `/`: endpoint tạo request thành công hoặc lỗi giả lập.
- `/healthz`: readiness probe.
- `/metrics`: Prometheus scrape metric.

Biến môi trường:

- `VERSION`: version hiện tại của API, ví dụ `v1`, `v2`.
- `ERROR_RATE`: tỷ lệ lỗi giả lập, ví dụ `0`, `0.2`, `0.5`.

Build image local cho Minikube:

```bash
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9
```

## Prometheus Và Alertmanager

Prometheus stack được cài bằng Helm qua Argo CD:

```text
argocd/apps/kube-prometheus-stack.yaml
```

Cấu hình quan trọng:

- `serviceMonitorSelectorNilUsesHelmValues: false`: cho phép Prometheus chọn ServiceMonitor ngoài Helm release mặc định.
- `ruleSelectorNilUsesHelmValues: false`: cho phép Prometheus chọn PrometheusRule ngoài Helm release mặc định.
- SMTP Alertmanager: gửi alert qua Gmail.

Kiểm tra Prometheus:

```bash
kubectl -n monitoring get pods
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

Mở:

```text
http://localhost:9090/targets
```

Kỳ vọng:

- Target của API ở trạng thái `UP`.
- Query metric có dữ liệu tăng theo thời gian.

## ServiceMonitor

File:

```text
k8s-api/servicemonitor.yaml
```

Prometheus scrape API qua:

- Service label: `app=api`
- Port: `http`
- Path: `/metrics`
- Interval: `15s`

Query kiểm tra dữ liệu:

```promql
flask_http_request_total{namespace="demo"}
```

## SLO Và AnalysisTemplate

File:

```text
k8s-api/analysis.yaml
```

SLO hiện cấu hình:

```text
Success Rate >= 95%
```

PromQL hiện dùng:

```promql
sum(rate(flask_http_request_duration_seconds_count{status!~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
```

Ý nghĩa:

- Tử số: request không phải lỗi `5xx`.
- Mẫu số: tổng request.
- Kết quả: tỷ lệ request thành công trong cửa sổ 1 phút.

Ngưỡng trong `AnalysisTemplate`:

```yaml
interval: 10s
successCondition: result >= 0.95
failureLimit: 2
```

Nghĩa là:

- Cứ 10 giây Prometheus được query một lần.
- Nếu success rate đạt từ 95% trở lên, bản canary được xem là khỏe.
- Nếu dưới 95% quá 2 lần, Rollout tự abort.

Ghi chú: nếu muốn dùng SLO 99%, đổi `successCondition` thành `result >= 0.99` và cập nhật rule alert tương ứng.

## Canary Tự Động

File:

```text
k8s-api/api.yaml
```

Chiến lược Canary:

```yaml
strategy:
  canary:
    analysis:
      templates:
        - templateName: success-rate-check
    steps:
      - setWeight: 25
      - pause:
          duration: 1m
      - setWeight: 50
      - pause:
          duration: 30s
      - setWeight: 100
```

Ý nghĩa:

- Đưa 25% traffic sang bản mới.
- Chạy analysis trong thời gian pause 1 phút.
- Nếu ổn, tăng lên 50%.
- Chạy tiếp analysis trong 30 giây.
- Nếu vẫn ổn, promote lên 100%.
- Nếu metric xấu, Rollout tự abort về bản ổn định trước đó.

Theo dõi Rollout:

```bash
kubectl argo rollouts get rollout api -n demo --watch
```

## Alert SLO

File:

```text
k8s-api/alerts.yaml
```

Alert `ApiHighErrorRate` sẽ fire khi success rate tụt dưới ngưỡng SLO.

Kiểm tra alert trong Prometheus:

```text
http://localhost:9090/alerts
```

Kỳ vọng khi inject lỗi:

- Alert chuyển sang trạng thái `Pending`.
- Sau thời gian `for`, alert chuyển sang `Firing`.
- Alertmanager gửi email đến địa chỉ đã cấu hình.

## Cách Demo Auto-Abort

1. Đảm bảo bản ổn định đang chạy:

```yaml
env:
  - name: ERROR_RATE
    value: '0'
  - name: VERSION
    value: 'v1'
```

2. Commit một bản lỗi:

```yaml
env:
  - name: ERROR_RATE
    value: '0.5'
  - name: VERSION
    value: 'v2'
```

3. Push lên Git:

```bash
git add k8s-api/api.yaml
git commit -m "test: inject api error for canary"
git push
```

4. Theo dõi Rollout:

```bash
kubectl argo rollouts get rollout api -n demo --watch
```

Kỳ vọng:

- Rollout bắt đầu canary.
- Metric lỗi tăng trong Prometheus.
- `AnalysisTemplate` fail.
- Rollout tự abort.
- ReplicaSet stable vẫn là bản `v1`.

## Rollback Bằng Git Revert

Rollback đúng theo GitOps:

```bash
git revert HEAD --no-edit
git push
```

Không dùng:

```bash
kubectl rollout undo
```

Lý do: nếu rollback trực tiếp trong cluster, Git vẫn giữ desired state lỗi. Argo CD sẽ self-heal cluster quay lại trạng thái trong Git.

Tiêu chí bài yêu cầu rollback bằng `git revert` dưới 5 phút.

## Ảnh Minh Chứng Cần Thêm

Tạo thư mục:

```text
docs/images/
```

Sau đó thêm các ảnh bên dưới. Tên file nên giữ đúng để README dễ kiểm tra.

| Tên ảnh | Mô tả cần chụp |
| --- | --- |
| `docs/images/01-argocd-apps-synced.png` | Màn hình Argo CD hoặc lệnh `kubectl -n argocd get applications`, thể hiện `root`, `api`, `kube-prometheus-stack`, `argo-rollouts` đều `Synced/Healthy`. |
| `docs/images/02-monitoring-pods-running.png` | Kết quả `kubectl -n monitoring get pods`, thể hiện Prometheus/Grafana/Alertmanager đang Running. |
| `docs/images/03-rollouts-controller-running.png` | Kết quả `kubectl -n argo-rollouts get pods`, thể hiện Argo Rollouts controller đang Running. |
| `docs/images/04-prometheus-target-api-up.png` | Trang Prometheus Targets, target API ở trạng thái `UP`. |
| `docs/images/05-prometheus-api-metric.png` | Query `flask_http_request_total{namespace="demo"}` hoặc query success rate có dữ liệu tăng. |
| `docs/images/06-rollout-stable-v1.png` | Kết quả `kubectl argo rollouts get rollout api -n demo`, thể hiện bản `v1` ổn định trước khi inject lỗi. |
| `docs/images/07-rollout-canary-running.png` | Rollout đang chạy canary sau khi đổi `VERSION=v2` và tăng `ERROR_RATE`. |
| `docs/images/08-rollout-auto-aborted.png` | Rollout tự abort khi analysis fail, thể hiện bản lỗi không lên 100%. |
| `docs/images/09-prometheus-alert-firing.png` | Trang Prometheus Alerts, alert `ApiHighErrorRate` ở trạng thái `Firing`. |
| `docs/images/10-email-alert-received.png` | Email cá nhân nhận được alert từ Alertmanager. Có thể che bớt thông tin nhạy cảm. |
| `docs/images/11-git-revert-rollback.png` | Terminal hoặc Git log thể hiện đã dùng `git revert` và push rollback thành công. |
| `docs/images/12-ci-validate-passed.png` | GitHub Actions job `validate` pass sau khi chạy kubeconform. |

Sau khi thêm ảnh, có thể nhúng vào README bằng cú pháp:

```md
![Argo CD apps synced](docs/images/01-argocd-apps-synced.png)
```

## Checklist Nộp Bài

- Repo có `Rollout`, `AnalysisTemplate`, `ServiceMonitor`, `PrometheusRule`.
- Toàn bộ thay đổi deploy qua Git và Argo CD.
- Argo CD app `api` synced, không drift.
- Prometheus scrape được API.
- Có SLO và giải thích rõ query/ngưỡng.
- Alert fire khi inject lỗi.
- Email alert gửi về email cá nhân.
- Canary bản lỗi tự abort.
- Rollback bằng `git revert` dưới 5 phút.
- README có ảnh hoặc mô tả ảnh minh chứng.

## Lệnh Kiểm Tra Nhanh

```bash
kubectl -n argocd get applications
kubectl -n monitoring get pods
kubectl -n argo-rollouts get pods
kubectl -n demo get rollout,pod,svc
kubectl argo rollouts get rollout api -n demo
```

Port-forward Prometheus:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

Mở:

```text
http://localhost:9090
```
