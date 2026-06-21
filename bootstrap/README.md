# Bootstrap

One-time setup steps applied manually before ArgoCD takes over. Everything in here is applied once — after that ArgoCD manages the cluster via the `apps/` directory.

---

## ArgoCD

Installed via Helm directly. After initial install, ArgoCD manages itself through the `app-of-apps` Application.

### Initial Install

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd -f bootstrap/argocd/values.yaml
```

### Get Initial Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### Access UI (before LoadBalancer is available)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
# Open http://localhost:8080
```

### Reinstalling ArgoCD

If ArgoCD needs to be reinstalled from scratch:

```bash
helm uninstall argocd -n argocd
helm install argocd argo/argo-cd -n argocd -f bootstrap/argocd/values.yaml
```

Then re-add the GitHub deploy key under Settings → Repositories and re-apply the root app:

```bash
kubectl apply -f apps/app-of-apps.yaml
```

### Upgrading ArgoCD

ArgoCD upgrades itself via the `apps/argocd/` Application once that is set up. To upgrade manually before that is in place:

```bash
helm repo update
helm upgrade argocd argo/argo-cd -n argocd -f bootstrap/argocd/values.yaml
```
