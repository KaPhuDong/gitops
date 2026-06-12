# Giải Thích Cấu Hình

Tài liệu này giải thích các phần cấu hình mentor có thể hỏi khi review bài. Mỗi mục gắn trực tiếp với file manifest trong repo và nội dung đã học: GitOps, CI/CD, observability, canary, alert và rollback.

## 1. Root App Và App-Of-Apps

File:

```text
argocd/root.yaml
```

Mục đích:

- `root` là Argo CD Application cha.
- `root` theo dõi thư mục `argocd/apps/`.
- Mỗi file YAML trong `argocd/apps/` định nghĩa một Application con.

Vì sao quan trọng:

- Chỉ cần apply `root` thủ công một lần.
- Sau đó muốn thêm app mới thì chỉ cần thêm file trong `argocd/apps/` và push Git.
- Đây là pattern app-of-apps.

Mentor có thể hỏi:

Tại sao không apply từng app thủ công?

Trả lời:

Apply thủ công khó scale và làm yếu GitOps. Với app-of-apps, Git vẫn là nguồn sự thật duy nhất, Argo CD tự tạo và cập nhật các app con.

## 2. Child Applications

Files:

```text
argocd/apps/api.yaml
argocd/apps/frontend.yaml
argocd/apps/backend.yaml
argocd/apps/web.yaml
argocd/apps/kube-prometheus-stack.yaml
argocd/apps/argo-rollouts.yaml
```

Mục đích:

- Mỗi file là một đơn vị deploy riêng.
- Mỗi Application bật automated sync.
- `frontend`, `backend`, `web` dùng `directory.include` để chỉ sync một file cụ thể trong `k8s/`.

Ví dụ:

```yaml
source:
  path: k8s
  directory:
    include: frontend.yaml
```

Vì sao quan trọng:

- Nếu không dùng `directory.include`, nhiều Argo CD app có thể cùng quản lý trùng resource trong cùng thư mục.
- Điều đó dễ gây lỗi ownership và trạng thái sync khó hiểu.

Mentor có thể hỏi:

Tại sao `frontend` không trỏ sang folder riêng?

Trả lời:

Repo này gom các manifest app đơn giản trong `k8s/`, nhưng `directory.include` giới hạn mỗi Application chỉ lấy đúng file của nó. Đây là cách nhẹ hơn so với tách nhiều folder.

## 3. Sync Waves

Files:

```text
k8s/frontend.yaml
k8s-api/api.yaml
argocd/apps/*.yaml
```

Mục đích:

- Sync wave điều khiển thứ tự apply.
- Wave nhỏ hơn chạy trước.

Ví dụ:

```yaml
annotations:
  argocd.argoproj.io/sync-wave: '1'
```

Frontend đang dùng:

```text
ConfigMap wave 1
Service wave 2
Deployment wave 2
```

Vì sao quan trọng:

- ConfigMap cần tồn tại trước khi Deployment mount nó.
- Monitoring stack và rollout controller nên sẵn sàng trước khi resource phụ thuộc vào chúng được apply.

Mentor có thể hỏi:

Không có sync wave thì sao?

Trả lời:

Kubernetes cuối cùng vẫn có thể converge, nhưng lần sync đầu dễ lỗi tạm thời nếu Deployment tham chiếu ConfigMap, CRD hoặc controller chưa sẵn sàng. Sync wave làm thứ tự apply rõ ràng hơn.

## 4. Frontend Service Và Proxy

File:

```text
k8s/frontend.yaml
```

Resources:

```text
ConfigMap/fe-page
ConfigMap/fe-nginx-config
Service/fe-svc
Deployment/fe
```

Mục đích:

- `fe-page` chứa HTML/CSS/JS của UI.
- `fe-nginx-config` cấu hình nginx reverse proxy.
- `fe-svc` expose FE trong cluster ở port `8081`.
- `Deployment/fe` chạy `nginx:alpine`.

Cấu hình nginx quan trọng:

```nginx
location /api/ {
  proxy_pass http://api.demo.svc.cluster.local:8080/;
}
```

Vì sao quan trọng:

- Browser gọi `/api/` cùng origin với frontend.
- nginx forward request đó tới API service nội bộ.
- Cách này tránh CORS và không cần expose API trực tiếp ra ngoài.

Mentor có thể hỏi:

Tại sao FE dùng `8081`?

Trả lời:

Argo CD UI thường port-forward local port `8080`. FE dùng `8081` để tránh xung đột:

```bash
kubectl -n demo port-forward svc/fe-svc 8081:8081
```

Mentor có thể hỏi:

FE có thật sự test API không?

Trả lời:

Có. Nút `Call API` gửi một request tới `/api/`. Nút `Burst 100 Calls` gửi 100 request theo batch. Các request đi qua nginx tới API service và làm metric Prometheus tăng.

## 5. API Rollout

File:

```text
k8s-api/api.yaml
```

Resource:

```text
Rollout/api
```

Mục đích:

- API không dùng Deployment thường.
- API dùng Argo Rollouts `Rollout`.
- Nhờ vậy có canary deployment và metric-based analysis.

Cấu hình quan trọng:

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
- Pause để Prometheus thu thập metric.
- Nếu analysis khỏe, tăng lên 50%.
- Nếu vẫn khỏe, promote lên 100%.
- Nếu analysis fail, rollout abort.

Mentor có thể hỏi:

Tại sao dùng Rollout thay vì Deployment?

Trả lời:

Deployment có rolling update nhưng không tự chạy analysis theo metric và không tự abort dựa trên Prometheus. Argo Rollouts bổ sung progressive delivery.

## 6. Inject Lỗi API

Files:

```text
app/app.py
k8s-api/api.yaml
```

Biến môi trường quan trọng:

```yaml
env:
  - name: ERROR_RATE
    value: '0'
  - name: VERSION
    value: 'v1'
```

Trong `app.py`:

```python
ERROR_RATE = float(os.getenv("ERROR_RATE", "0"))
VERSION = os.getenv("VERSION", "v1")
```

Ý nghĩa:

- `ERROR_RATE=0`: tất cả request gần như thành công.
- `ERROR_RATE=0.5`: khoảng 50% request trả HTTP 500.
- `VERSION`: chứng minh version nào đang chạy.

Mentor có thể hỏi:

Test alert và abort như thế nào?

Trả lời:

Đổi `ERROR_RATE` thành `0.5`, đổi `VERSION` thành `v2-bad`, commit, push, rồi bấm `Burst 100 Calls` trên frontend để tạo traffic lỗi.

## 7. API Metrics

Files:

```text
app/app.py
app/Dockerfile
```

Code quan trọng:

```python
from prometheus_flask_exporter import PrometheusMetrics
PrometheusMetrics(app)
```

Mục đích:

- Tự expose endpoint `/metrics`.
- Tự tạo HTTP request metrics theo status code và endpoint.

Metric quan trọng:

```promql
flask_http_request_duration_seconds_count{namespace="demo"}
```

Mentor có thể hỏi:

Tại sao dùng `flask_http_request_duration_seconds_count` thay vì custom metric?

Trả lời:

`prometheus-flask-exporter` tự cung cấp metric này cho mọi request Flask. Nó đủ để tính total traffic, success rate và error rate.

## 8. ServiceMonitor

File:

```text
k8s-api/servicemonitor.yaml
```

Mục đích:

- Nói cho Prometheus biết scrape API ở đâu.

Cấu hình quan trọng:

```yaml
selector:
  matchLabels:
    app: api
endpoints:
  - port: http
    path: /metrics
    interval: 15s
```

Ý nghĩa:

- Tìm Service có label `app=api`.
- Scrape named port `http`.
- Scrape path `/metrics`.
- Scrape mỗi 15 giây.

Mentor có thể hỏi:

Tại sao scrape Service thay vì Pod IP?

Trả lời:

Pod IP thay đổi liên tục. Service ổn định hơn, và ServiceMonitor tích hợp với Prometheus Operator để tự động discover endpoint.

## 9. AnalysisTemplate

File:

```text
k8s-api/analysis.yaml
```

Mục đích:

- Định nghĩa metric check cho Argo Rollouts.
- Query Prometheus trong quá trình canary.

Cấu hình quan trọng:

```yaml
interval: 10s
successCondition: result >= 0.95
failureLimit: 2
```

PromQL:

```promql
sum(rate(flask_http_request_duration_seconds_count{status!~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
```

Ý nghĩa:

- Tử số: request thành công, tức status không phải 5xx.
- Mẫu số: tổng request API.
- Kết quả: success rate.
- Nếu success rate >= 95%, canary khỏe.

Mentor có thể hỏi:

Tại sao dùng cửa sổ `[1m]`?

Trả lời:

Cửa sổ 1 phút làm mượt spike ngắn nhưng vẫn phản ứng đủ nhanh cho demo. Rollout cũng pause 1 phút và 30 giây, nên cửa sổ này hợp lý.

Mentor có thể hỏi:

`failureLimit: 2` nghĩa là gì?

Trả lời:

Analysis được phép fail 2 lần trước khi rollout bị coi là fail. Điều này tránh abort chỉ vì một sample nhiễu.

## 10. PrometheusRule

File:

```text
k8s-api/alerts.yaml
```

Mục đích:

- Định nghĩa alert SLO cho API.

Cấu hình quan trọng:

```yaml
expr: |
  (
    sum(rate(flask_http_request_duration_seconds_count{status=~"5..", namespace="demo"}[1m]))
    /
    sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
  ) > 0.05
for: 30s
labels:
  severity: critical
```

Ý nghĩa:

- Nếu hơn 5% request là 5xx trong 30 giây, fire alert critical.

Mentor có thể hỏi:

PrometheusRule liên quan gì tới AnalysisTemplate?

Trả lời:

Cả hai đều dùng metric Prometheus. AnalysisTemplate phục vụ Argo Rollouts để quyết định promote/abort canary. PrometheusRule phục vụ Prometheus/Alertmanager để báo cho con người.

## 11. Alertmanager Email

File:

```text
argocd/apps/kube-prometheus-stack.yaml
```

Cấu hình quan trọng:

```yaml
alertmanager:
  config:
    global:
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'kaphudong04@gmail.com'
      smtp_auth_username: 'kaphudong04@gmail.com'
      smtp_auth_password: 'TODO_GMAIL_APP_PASSWORD'
    route:
      group_by: ['alertname']
      receiver: 'email-ca-nhan'
    receivers:
      - name: 'null'
      - name: 'email-ca-nhan'
        email_configs:
          - to: 'kaphudong04@gmail.com'
            send_resolved: true
```

Vì sao có receiver `null`:

- kube-prometheus-stack có route mặc định đưa một số alert như Watchdog tới receiver `null`.
- Nếu override receivers mà thiếu `null`, Alertmanager có thể lỗi `undefined receiver "null"`.

Mentor có thể hỏi:

Tại sao không commit Gmail App Password thật?

Trả lời:

Đó là secret. Password thật nên đưa qua Kubernetes Secret, sealed secret, external secret hoặc private values file, không commit vào repo public.

## 12. Selector Của kube-prometheus-stack

File:

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

- Prometheus có thể chọn ServiceMonitor và PrometheusRule ngoài default Helm release.
- Namespace selector `{}` cho phép tìm qua nhiều namespace.
- Cần cấu hình này vì API monitoring resource nằm ở namespace `demo`, còn Prometheus nằm ở namespace `monitoring`.

Mentor có thể hỏi:

Tại sao ServiceMonitor và PrometheusRule nằm ở namespace `demo`?

Trả lời:

Chúng thuộc API app và được deploy cùng API. Prometheus được cấu hình để discover monitoring resource cross-namespace.

## 13. EndpointSlice

Files:

```text
argocd/apps/kube-prometheus-stack.yaml
k8s-api/servicemonitor.yaml
```

Mục đích:

- Kubernetes v1.33+ cảnh báo `Endpoints` cũ đã deprecated.
- `EndpointSlice` là API discovery mới hơn.

Cấu hình:

```yaml
serviceDiscoveryRole: EndpointSlice
```

Mentor có thể hỏi:

Warning `Endpoints deprecated` có fatal không?

Trả lời:

Không. Đó là warning, không phải lỗi làm app fail. Nhưng dùng EndpointSlice phù hợp hơn với Kubernetes version mới.

## 14. Rollback

Lệnh rollback:

```bash
git revert HEAD --no-edit
git push
```

Vì sao quan trọng:

- Git vẫn là source of truth.
- Argo CD thấy revert commit và reconcile cluster.
- Git history thể hiện ai rollback và rollback lúc nào.

Mentor có thể hỏi:

Tại sao không dùng `kubectl rollout undo`?

Trả lời:

Lệnh đó chỉ sửa cluster. Git vẫn giữ desired state lỗi, nên Argo CD có thể sync bản lỗi quay lại.

## 15. CI Validate

Workflow chạy:

```bash
kubeconform -strict -ignore-missing-schemas -summary argocd/ k8s/ k8s-api/
```

Mục đích:

- Validate YAML và manifest trước khi merge.
- Bắt lỗi sớm.

Ví dụ lỗi:

```yaml
image: image: w9-api:1
```

Dòng này lỗi vì có hai dấu mapping `:` trên cùng một dòng.

Dạng đúng:

```yaml
image: w9-api:1
```

Mentor có thể hỏi:

Tại sao dùng `ignore-missing-schemas`?

Trả lời:

Một số CRD như Argo CD Application, Argo Rollouts, ServiceMonitor, PrometheusRule có thể không có schema mặc định trong kubeconform. Ta vẫn validate YAML và resource built-in, đồng thời bỏ qua schema CRD bị thiếu.

## 16. UI Chứng Minh Được Gì

Frontend chứng minh:

- FE được deploy bằng GitOps.
- FE gọi API qua Kubernetes service DNS.
- Người dùng tương tác trực tiếp để tạo API traffic.
- Prometheus metric tăng sau khi bấm nút.
- Burst traffic hỗ trợ test canary/alert mà không cần vòng lặp curl thủ công.

UI không tự chứng minh được:

- UI không tự tạo lỗi nếu `ERROR_RATE=0`.
- Alert cần API trả 5xx.
- Canary abort cần deploy một bad version và có đủ traffic.

Để chứng minh alert/abort đầy đủ:

1. Set API `ERROR_RATE=0.5`.
2. Set `VERSION=v2-bad`.
3. Commit và push.
4. Bấm `Burst 100 Calls`.
5. Quan sát Rollout và Prometheus Alerts.

## Quick Mentor Q&A

Q: Source of truth là gì?

A: Manifest trong GitHub repo. Argo CD reconcile cluster để khớp Git.

Q: Argo CD làm gì?

A: So sánh desired state trong Git với live state trong Kubernetes và sync khi lệch.

Q: Prometheus làm gì?

A: Scrape `/metrics` từ API và đánh giá SLO query/alert.

Q: Alertmanager làm gì?

A: Nhận alert firing từ Prometheus và route notification, ví dụ gửi email.

Q: Argo Rollouts làm gì?

A: Quản lý progressive delivery và dùng Prometheus analysis để promote hoặc abort canary.

Q: Tại sao FE dùng nginx?

A: nginx vừa serve static UI vừa proxy `/api/` tới API service nội bộ, giúp browser chỉ cần gọi cùng một origin.

Q: SLO success là bao nhiêu?

A: Ít nhất 95% API request phải không phải 5xx.

Q: Khi nào alert fire?

A: Khi API 5xx error rate vượt 5% trong 30 giây.

Q: Rollback an toàn như thế nào?

A: Dùng `git revert`, push lên Git, để Argo CD sync desired state đã revert.
