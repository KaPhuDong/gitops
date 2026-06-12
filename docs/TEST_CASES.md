# Hướng Dẫn Test Các Case

Tài liệu này dùng để kiểm tra end-to-end cho dự án Ship Smartly: GitOps, observability, canary analysis, alerting, rollback và frontend tương tác trực tiếp với API.

## Điều Kiện Chuẩn Bị

- Kubernetes context đang trỏ đúng cluster demo.
- Argo CD đã cài trong namespace `argocd`.
- Root Application đã được apply một lần:

```bash
kubectl apply -f argocd/root.yaml
```

- Nếu dùng image local trên Minikube, image API đã được load vào cluster:

```bash
docker build -t w9-api:1 app/
minikube image load w9-api:1 -p w9
```

## Port Forward

Argo CD UI:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Frontend UI:

```bash
kubectl -n demo port-forward svc/fe-svc 8081:8081
```

Prometheus:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090
```

Grafana:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80
```

Alertmanager:

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093
```

## Case 1: Argo CD Apps Đều Synced/Healthy

Mục tiêu: chứng minh toàn bộ workload được quản lý bằng Argo CD và đồng bộ từ Git.

Chạy:

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

Minh chứng nên chụp:

- Màn hình danh sách app trong Argo CD.
- Terminal chạy `kubectl -n argocd get applications`.

## Case 2: Pod Và Service Đang Chạy

Mục tiêu: chứng minh runtime workload đã sẵn sàng.

Chạy:

```bash
kubectl -n demo get rollout,pod,svc
kubectl -n monitoring get pods
kubectl -n argo-rollouts get pods
```

Kỳ vọng:

- `rollout/api` có 4 desired/current/available replicas.
- Pod `fe-*`, `api-*`, `backend-*`, `web-*` ở trạng thái Running.
- Prometheus, Alertmanager, Grafana và Prometheus Operator ở trạng thái Running.
- Argo Rollouts controller ở trạng thái Running.

Minh chứng nên chụp:

- Output terminal.
- Resource tree trong Argo CD hiển thị Healthy.

## Case 3: Frontend Gọi API Trực Tiếp

Mục tiêu: chứng minh UI có tương tác thật với API, không cần gọi curl thủ công.

Mở:

```text
http://localhost:8081
```

Bấm:

```text
Call API
```

Kỳ vọng:

- Khung kết quả trên UI hiển thị JSON tương tự:

```json
{
  "status": 200,
  "ok": true,
  "body": {
    "ok": true,
    "version": "v1"
  }
}
```

Kiểm tra trực tiếp trong cluster:

```bash
kubectl -n demo exec deploy/fe -- wget -qO- http://127.0.0.1/api/
```

Kỳ vọng:

```json
{"ok":true,"version":"v1"}
```

Minh chứng nên chụp:

- Màn hình frontend sau khi bấm `Call API`.
- Output kiểm tra trực tiếp từ pod FE.

## Case 4: Frontend Burst Tạo Metric Cho Prometheus

Mục tiêu: chứng minh UI tạo traffic thật để phục vụ observability và rollout analysis.

Mở:

```text
http://localhost:8081
```

Bấm:

```text
Burst 100 Calls
```

Sau đó mở Prometheus:

```text
http://localhost:9090
```

Query:

```promql
sum(flask_http_request_duration_seconds_count{namespace="demo"})
```

Kỳ vọng:

- Tổng số request tăng sau khi bấm `Burst 100 Calls`.
- Frontend hiển thị số total/success/error.

Query success rate:

```promql
sum(rate(flask_http_request_duration_seconds_count{status!~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
```

Kỳ vọng khi API đang khỏe:

```text
1
```

Minh chứng nên chụp:

- Màn hình FE sau khi burst.
- Prometheus query trước và sau burst.

## Case 5: Prometheus Scrape Được API

Mục tiêu: chứng minh `ServiceMonitor` hoạt động.

Mở:

```text
http://localhost:9090/targets
```

Kỳ vọng:

- Target API ở trạng thái `UP`.
- Target được sinh từ `ServiceMonitor/api-monitor`.

Kiểm tra CLI:

```bash
kubectl -n demo get servicemonitor api-monitor -o yaml
kubectl -n demo get prometheusrule api-slo-alerts -o yaml
```

Minh chứng nên chụp:

- Trang Prometheus Targets.
- Output `ServiceMonitor`.

## Case 6: Canary Tốt Được Promote Thành Công

Mục tiêu: chứng minh bản khỏe được rollout lên 100%.

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

Theo dõi rollout:

```bash
kubectl argo rollouts get rollout api -n demo --watch
```

Tạo traffic từ FE:

```text
http://localhost:8081 -> Burst 100 Calls
```

Kỳ vọng:

- Rollout đi qua 25%, 50%, rồi 100%.
- Analysis pass vì success rate >= 95%.
- Rollout trở về Healthy.

Minh chứng nên chụp:

- Output rollout watch.
- Stats trên FE.
- Prometheus success-rate query.

## Case 7: Canary Lỗi Tự Abort

Mục tiêu: chứng minh Argo Rollouts tự abort bản lỗi dựa trên metric Prometheus.

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

Trong lúc canary đang chạy, tạo traffic từ FE:

```text
http://localhost:8081 -> Burst 100 Calls
```

Prometheus query:

```promql
sum(rate(flask_http_request_duration_seconds_count{status=~"5..", namespace="demo"}[1m]))
/
sum(rate(flask_http_request_duration_seconds_count{namespace="demo"}[1m]))
```

Kỳ vọng:

- Error rate vượt `0.05`.
- `AnalysisTemplate/success-rate-check` fail sau `failureLimit`.
- Rollout abort và giữ stable ReplicaSet.
- Bản lỗi không được promote lên 100%.

Minh chứng nên chụp:

- Rollout hiển thị `Aborted` hoặc analysis failure.
- Prometheus query cho thấy error rate tăng.
- FE burst stats có lỗi.

## Case 8: Alert API Error Rate Firing

Mục tiêu: chứng minh `PrometheusRule` và Alertmanager hoạt động.

Giữ bản lỗi ở Case 7 đủ lâu để tạo lỗi.

Mở:

```text
http://localhost:9090/alerts
```

Kỳ vọng:

- Alert `ApiSuccessRateSLIViolation` chuyển sang `Pending`.
- Sau `for: 30s`, alert chuyển sang `Firing`.

Query CLI trong Prometheus:

```promql
ALERTS{alertname="ApiSuccessRateSLIViolation"}
```

Kỳ vọng:

```text
alertstate="pending" hoặc alertstate="firing"
```

Minh chứng nên chụp:

- Trang Prometheus Alerts.
- Alertmanager UI hiển thị alert.
- Email nhận tại `kaphudong04@gmail.com`.

Lưu ý:

- Gửi email cần Gmail App Password thật trong cấu hình Alertmanager.
- Không commit app password thật lên repo public.

## Case 9: Rollback Bằng Git Revert

Mục tiêu: chứng minh rollback đi qua Git, không sửa tay trong cluster.

Rollback commit lỗi:

```bash
git revert HEAD --no-edit
git push
```

Theo dõi Argo CD và Rollout:

```bash
kubectl -n argocd get application api
kubectl argo rollouts get rollout api -n demo --watch
```

Kỳ vọng:

- Argo CD sync desired state đã revert.
- API trở về version khỏe trước đó.
- Git history có commit revert.

Minh chứng nên chụp:

- `git log --oneline`.
- Argo CD application history.
- Rollout Healthy sau revert.

## Case 10: CI Validate Pass

Mục tiêu: chứng minh manifest không lỗi YAML/schema cơ bản.

GitHub Actions chạy:

```bash
kubeconform -strict -ignore-missing-schemas -summary argocd/ k8s/ k8s-api/
```

Kỳ vọng:

- Workflow màu xanh.
- Không có lỗi YAML parse.

Nếu local không có `kubeconform`, có thể kiểm tra tạm:

```bash
kubectl apply --dry-run=client --validate=false -f argocd/root.yaml
kubectl apply --dry-run=client --validate=false -f k8s/
kubectl apply --dry-run=client --validate=false -f k8s-api/
```

Minh chứng nên chụp:

- GitHub Actions job `validate` pass.

## Checklist Nộp Bài

- Argo CD apps Synced/Healthy.
- FE gọi API trực tiếp được.
- FE Burst tạo request traffic.
- Prometheus target API ở trạng thái UP.
- Prometheus metric tăng sau burst.
- Good canary promote lên 100%.
- Bad canary auto-abort.
- API alert firing.
- Alertmanager gửi email.
- Rollback bằng `git revert`.
- CI validate pass.
