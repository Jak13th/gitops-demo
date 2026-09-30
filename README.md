# Demo Gitops for kubernetes + ArgoCD


# Useful cmd


## ArgoCD

**port-forward for ArgoCD UI:**


```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```



```bash
git add argocd/ && git commit -m "Application ArgoCD hello" && git push
kubectl apply -f argocd/hello-app.yaml
```