# Challenge: Ship Smartly

Repo này triển khai bài Challenge **Ship Smartly** theo 3 phần chính:

- **GitOps**: mọi thay đổi đi qua Git, Argo CD tự sync và self-heal.
- **Observability**: Prometheus/Grafana thu thập metric, có SLO và alert.
- **Progressive Delivery**: API dùng Argo Rollouts Canary, bản lỗi tự abort dựa trên metric Prometheus.

## Mục Tiêu Cần Đạt

- Argo CD quản lý toàn bộ manifest theo mô hình App-of-Apps.
- Frontend có UI tương tác trực tiếp với API.
- API có endpoint `/metrics` để Prometheus scrape.
- Có SLO success rate và alert gửi email.
- Canary dùng `AnalysisTemplate` để tự chấm điểm bằng Prometheus.
- Bản tốt được promote lên 100%, bản lỗi tự abort về bản ổn định.
- Rollback bằng `git revert`, không rollback tay trong cluster.
- CI validate manifest bằng `kubeconform`.

## Cấu Trúc Repo

```text
gitops/
|-- argocd/
|   |-- root.yaml
|   `-- apps/
|       |-- api.yaml
|       |-- argo-rollouts.yaml
|       |-- backend.yaml
|       |-- frontend.yaml
|       |-- kube-prometheus-stack.yaml
|       `-- web.yaml
|-- app/
|   |-- app.py
|   `-- Dockerfile
|-- docs/
|   |-- CONFIG_EXPLANATION.md
|   |-- PROJECT_FLOW.md
|   `-- TEST_CASES.md
|-- k8s/
|   |-- namespace.yaml
|   |-- backend.yaml
|   |-- frontend.yaml
|   `-- web.yaml
`-- k8s-api/
    |-- alerts.yaml
    |-- analysis.yaml
    |-- api.yaml
    `-- servicemonitor.yaml
```

## Tài Liệu Bổ Sung

- [docs/TEST_CASES.md](docs/TEST_CASES.md): hướng dẫn test từng case end-to-end.
- [docs/PROJECT_FLOW.md](docs/PROJECT_FLOW.md): giải thích flow dự án từ Git đến cluster, metric, alert và rollback.
- [docs/CONFIG_EXPLANATION.md](docs/CONFIG_EXPLANATION.md): giải thích cấu hình mentor có thể hỏi khi review bài.

## App-Of-Apps

Root Application:

```text
argocd/root.yaml
```

Root app trỏ tới:

```text
argocd/apps/
```

Các app con:

- `api`: deploy toàn bộ `k8s-api/`.
- `frontend`: deploy `k8s/frontend.yaml`.
- `backend`: deploy `k8s/backend.yaml`.
- `web`: deploy `k8s/web.yaml`.
- `kube-prometheus-stack`: cài Prometheus, Grafana, Alertmanager.
- `argo-rollouts`: cài Argo Rollouts controller.

Các app `frontend`, `backend`, `web` dùng `directory.include` để mỗi Application chỉ sync đúng file của nó trong thư mục `k8s/`.

Ví dụ:

```yaml
source:
  path: k8s
  directory:
    include: frontend.yaml
```

## Deploy Bằng GitOps

Apply root Application một lần:

```bash
kubectl apply -f argocd/root.yaml
```

Kiểm tra Argo CD:

```bash
kubectl -n argocd get applications
```

Kỳ vọng:

```text
root                    Synced   Healthy
api                     Synced   Healthy
frontend                Synced   Healthy
kube-prometheus-stack   Synced   Healthy
argo-rollouts           Synced   Healthy
backend                 Synced   Healthy
web                     Synced   Healthy
```

## Frontend UI

Frontend nằm ở:

```text
k8s/frontend.yaml
```

Frontend gồm:

- `ConfigMap/fe-page`: chứa HTML/CSS/JS.
- `ConfigMap/fe-nginx-config`: cấu hình nginx proxy.
- `Deployment/fe`: chạy `nginx:alpine`.
- `Service/fe-svc`: expose port `8081`.

Port-forward FE:

```bash
kubectl -n demo port-forward svc/fe-svc 8081:8081
```

Mở:

```text
http://localhost:8081
```

UI có 2 nút chính:

- `Call API`: gọi API một lần và hiển thị JSON response.
- `Burst 100 Calls`: gọi API nhiều lần để tạo traffic cho Prometheus, AnalysisTemplate và alert.

Frontend gọi API qua nginx proxy:

```text
Browser -> /api/ -> nginx -> http://api.demo.svc.cluster.local:8080/
```

Cách này giúp browser gọi same-origin, tránh CORS và không cần expose API trực tiếp.

## API

Source API:

```text
app/app.py
app/Dockerfile
```

API endpoints:

- `/`: trả JSON success hoặc lỗi giả lập.
- `/healthz`: readiness probe.
- `/metrics`: Prometheus scrape metric.

Biến môi trường quan trọng:

```yaml
env:
  - name: ERROR_RATE
    value: '0'
  - name: VERSION
    value: 'v1'
```

Ý nghĩa:

- `ERROR_RATE=0`: API chạy khỏe.
- `ERROR_RATE=0.5`: khoảng 50% request trả HTTP 500.
- `VERSION`: giúp chứng minh version đang chạy.

Build image local cho Minikube:

```bash
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9
```

## Prometheus Và Alertmanager

Prometheus stack được cài bằng Helm qua:

```text
argocd/apps/kube-prometheus-stack.yaml
```

Cấu hình quan trọng:

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorNamespaceSelector: {}
    ruleSelectorNilUsesHelmValues: false
    ruleNamespaceSelector: {}
```

Ý nghĩa:

- Prometheus có thể scrape `ServiceMonitor` ngoài Helm chart.
- Prometheus có thể đọc `PrometheusRule` ngoài Helm chart.
- Namespace selector `{}` cho phép đọc resource ở namespace `demo`.

Port-forward Prometheus:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

Mở:

```text
http://localhost:9090
```

Port-forward Alertmanager:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093
```

Mở:

```text
http://localhost:9093
```

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

Query kiểm tra tổng request:

```promql
sum(flask_http_request_duration_seconds_count{namespace="demo"})
```

## SLO Và AnalysisTemplate

File:

```text
k8s-api/analysis.yaml
```

SLO:

```text
Success rate >= 95%
```

PromQL success rate:

```promql
sum(rate(flask_http_request_duration_seconds_count{status!~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
```

Cấu hình:

```yaml
interval: 10s
successCondition: result[0] >= 0.95
failureLimit: 2
```

Ý nghĩa:

- Cứ 10 giây Argo Rollouts query Prometheus một lần.
- Nếu success rate >= 95%, canary được xem là khỏe.
- Nếu success rate dưới ngưỡng quá số lần cho phép, rollout abort.

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
- Pause để Prometheus có dữ liệu.
- Nếu analysis pass, tăng lên 50%.
- Nếu vẫn pass, promote lên 100%.
- Nếu metric xấu, rollout abort về bản ổn định.

Theo dõi rollout:

```bash
kubectl argo rollouts get rollout api -n demo --watch
```

## Alert SLO

File:

```text
k8s-api/alerts.yaml
```

Alert fire khi error rate > 5% trong 30 giây:

```promql
sum(rate(flask_http_request_duration_seconds_count{status=~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
```

Kiểm tra alert:

```text
http://localhost:9090/alerts
```

Query:

```promql
ALERTS{alertname="ApiSuccessRateSLIViolation"}
```

Email receiver đang cấu hình:

```text
kaphudong04@gmail.com
```

Lưu ý:

- `smtp_auth_password` vẫn nên để dạng secret, không commit password thật lên repo public.
- Nếu password còn là `TODO_GMAIL_APP_PASSWORD`, alert có thể firing nhưng email chưa gửi được.

## Demo Good Canary

Sửa `k8s-api/api.yaml`:

```yaml
env:
  - name: ERROR_RATE
    value: '0'
  - name: VERSION
    value: 'v2-good'
```

Commit và push:

```bash
git add k8s-api/api.yaml
git commit -m "test: deploy healthy api canary"
git push
```

Tạo traffic bằng FE:

```text
http://localhost:8081 -> Burst 100 Calls
```

Kỳ vọng:

- Success rate >= 95%.
- Analysis pass.
- Rollout promote lên 100%.

## Demo Bad Canary Auto-Abort

Sửa `k8s-api/api.yaml`:

```yaml
env:
  - name: ERROR_RATE
    value: '0.5'
  - name: VERSION
    value: 'v2-bad'
```

Commit và push:

```bash
git add k8s-api/api.yaml
git commit -m "test: inject api errors for canary"
git push
```

Theo dõi rollout:

```bash
kubectl argo rollouts get rollout api -n demo --watch
```

Tạo traffic bằng FE:

```text
http://localhost:8081 -> Burst 100 Calls
```

Kỳ vọng:

- FE hiển thị một phần request lỗi.
- Prometheus error rate > 5%.
- Alert `ApiSuccessRateSLIViolation` pending rồi firing.
- Analysis fail.
- Rollout tự abort, bản lỗi không lên 100%.

## Rollback Bằng Git Revert

Rollback đúng GitOps:

```bash
git revert HEAD --no-edit
git push
```

Không dùng làm cách rollback chính:

```bash
kubectl rollout undo
```

Lý do:

- `kubectl rollout undo` chỉ sửa cluster.
- Git vẫn giữ desired state lỗi.
- Argo CD có thể sync bản lỗi quay lại.

## CI Validate

GitHub Actions chạy:

```bash
kubeconform -strict -ignore-missing-schemas -summary argocd/ k8s/ k8s-api/
```

Mục tiêu:

- Bắt lỗi YAML.
- Validate manifest Kubernetes cơ bản.
- Bỏ qua schema CRD không có sẵn như Argo CD Application, Rollout, ServiceMonitor, PrometheusRule.

Local fallback nếu không có `kubeconform`:

```bash
kubectl apply --dry-run=client --validate=false -f argocd/root.yaml
kubectl apply --dry-run=client --validate=false -f k8s/
kubectl apply --dry-run=client --validate=false -f k8s-api/
```

## Lệnh Kiểm Tra Nhanh

```bash
kubectl -n argocd get applications
kubectl -n monitoring get pods
kubectl -n argo-rollouts get pods
kubectl -n demo get rollout,pod,svc
kubectl argo rollouts get rollout api -n demo
```

Kiểm tra FE gọi API:

```bash
kubectl -n demo exec deploy/fe -- wget -qO- http://127.0.0.1/api/
```

Kỳ vọng:

```json
{"ok":true,"version":"v1"}
```

## Checklist Nộp Bài

- Repo có `Rollout`, `AnalysisTemplate`, `ServiceMonitor`, `PrometheusRule`.
- Toàn bộ thay đổi deploy qua Git và Argo CD.
- Argo CD apps Synced/Healthy.
- FE UI gọi API trực tiếp được.
- FE Burst tạo traffic thật.
- Prometheus scrape được API.
- Prometheus metric tăng sau khi Burst.
- Có SLO và giải thích rõ query/ngưỡng.
- Alert fire khi inject lỗi.
- Alertmanager gửi email.
- Canary bản tốt promote.
- Canary bản lỗi auto-abort.
- Rollback bằng `git revert`.
- CI validate pass.

## Evidence

Phần này tổng hợp minh chứng đã thu thập trong quá trình triển khai và kiểm thử dự án. Các ảnh được sắp xếp theo luồng demo: GitOps sync, runtime health, frontend tương tác API, Prometheus metrics, canary analysis, alerting và CI validation.

### 1. GitOps Và Trạng Thái Cluster

#### 01. Argo CD Apps Synced/Healthy

Toàn bộ Application do Argo CD quản lý đều ở trạng thái `Synced/Healthy`, chứng minh desired state trong Git đã được reconcile vào cluster.

![Argo CD apps synced](docs/images/01-argocd-apps-synced.png)

#### 02. Demo Namespace Pods/Services Running

Các workload trong namespace `demo` đang chạy, bao gồm API Rollout, frontend, backend, web và service `fe-svc` dùng port `8081`.

![Demo pods running](docs/images/02-demo-pods-running.png)

#### 03. Monitoring Stack Running

Prometheus, Grafana, Alertmanager, kube-state-metrics và Prometheus Operator đang Running trong namespace `monitoring`.

![Monitoring pods running](docs/images/03-monitoring-pods-running.png)

#### 04. Argo Rollouts Controller Running

Argo Rollouts controller đã sẵn sàng để quản lý rollout canary và analysis run.

![Argo Rollouts controller running](docs/images/04-rollouts-controller-running.png)

### 2. Frontend Tương Tác API

#### 05. Frontend Gọi API Thành Công

Frontend gọi API qua nginx proxy `/api/` và hiển thị JSON response trực tiếp trên UI.

![Frontend call API](docs/images/05-frontend-call-api.png)

#### 06. Frontend Burst Tạo Traffic

Nút `Burst 100 Calls` tạo request traffic từ browser đến API, thay thế thao tác gọi curl thủ công khi cần tạo dữ liệu cho Prometheus và analysis.

![Frontend burst traffic](docs/images/06-frontend-burst-traffic.png)

### 3. Prometheus Metrics

#### 07. Prometheus Target API UP

Prometheus phát hiện API target thông qua `ServiceMonitor/api-monitor` và target ở trạng thái `UP`.

![Prometheus target API up](docs/images/07-prometheus-target-api-up.png)

#### 08. Request Count Tăng Sau Burst

Metric `flask_http_request_duration_seconds_count{namespace="demo"}` có dữ liệu và request count tăng sau khi tạo traffic từ frontend.

![Prometheus API request total](docs/images/08-prometheus-api-request-total.png)

#### 09. Success Rate Bản Khỏe

Prometheus query success rate cho bản khỏe cho kết quả gần hoặc bằng `1`, phù hợp với SLO success rate >= 95%.

![Prometheus success rate](docs/images/09-prometheus-success-rate.png)

### 4. Canary Và Analysis

#### 10. Good Canary Promote Thành Công

Bản canary khỏe được Argo Rollouts phân tích bằng Prometheus metric và promote thành công lên 100%.

![Good canary promoted](docs/images/10-good-canary-promoted.png)

#### 11. Bad Canary Auto-Abort

Khi inject lỗi API, analysis fail và Argo Rollouts abort bản canary lỗi, ngăn bản lỗi được promote lên 100%.

![Bad canary auto aborted](docs/images/12-bad-canary-auto-aborted.png)

### 5. Alerting

#### 12. Prometheus Error Rate Vượt Ngưỡng

Prometheus query error rate cho thấy tỷ lệ lỗi 5xx vượt ngưỡng `0.05` sau khi inject lỗi.

![Prometheus error rate](docs/images/13-prometheus-error-rate.png)

#### 13. Prometheus Alert Firing

Alert `ApiSuccessRateSLIViolation` chuyển sang trạng thái `Firing`, chứng minh `PrometheusRule` hoạt động.

![Prometheus alert firing](docs/images/14-prometheus-alert-firing.png)

#### 14. Email Alert Received

Alertmanager gửi email cảnh báo đến địa chỉ nhận đã cấu hình.

![Email alert received](docs/images/16-email-alert-received.png)

### 6. CI Validate

#### 15. GitHub Actions Validate Passed

GitHub Actions job `validate` chạy `kubeconform` và pass, chứng minh manifest không còn lỗi YAML/schema cơ bản.

![CI validate passed](docs/images/19-ci-validate-passed.png)
