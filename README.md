# GitOps Demo

Repository nay chua manifest Kubernetes va Argo CD Application de deploy ung dung `web` vao namespace `demo`.

## Cau truc

```text
.
+-- argocd/
|   +-- root.yaml          # App-of-apps root Application
|   +-- apps/
|       +-- web.yaml       # Argo CD Application cho ung dung web
+-- k8s/
|   +-- namespace.yaml     # Namespace demo
|   +-- web.yaml           # Deployment nginx
+-- .github/workflows/
    +-- validate.yml       # Validate manifest bang kubeconform tren pull request
```

## Dieu kien can co

- Kubernetes cluster dang chay.
- `kubectl` da cau hinh dung cluster.
- Argo CD da duoc cai trong namespace `argocd`.
- Repo URL trong `argocd/root.yaml` va `argocd/apps/web.yaml` tro ve dung GitHub repository.

## Deploy bang Argo CD

Apply root Application:

```bash
kubectl apply -f argocd/root.yaml
```

Root Application se doc cac app con trong `argocd/apps`. App `web` se sync manifest trong thu muc `k8s` va tao:

- Namespace `demo`
- Deployment `web`
- Pod chay image `nginx:1.27`

Kiem tra Application:

```bash
kubectl -n argocd get applications
kubectl -n argocd describe application root
kubectl -n argocd describe application web
```

Kiem tra workload:

```bash
kubectl -n demo get deploy,pod
```

## Deploy truc tiep bang kubectl

Dung cach nay neu muon test manifest Kubernetes khong thong qua Argo CD:

```bash
kubectl apply -f k8s/
kubectl -n demo get deploy,pod
```

## Validate manifest

GitHub Actions se chay `kubeconform` khi pull request thay doi file trong `k8s/**`.

Co the validate local bang:

```bash
kubectl apply --dry-run=client -f k8s/
```

## Luu y

Argo CD lay manifest tu Git repository, khong lay tu file local. Sau khi sua manifest, can commit va push len branch `main` de Argo CD co the sync:

```bash
git add .
git commit -m "Update manifests"
git push
```
