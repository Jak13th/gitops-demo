# Relancer la stack Kubernetes + ArgoCD from scratch

Séquence complète pour recréer le cluster local, installer ArgoCD et déployer l'app `hello` (Nginx + page HTML dans un ConfigMap), à partir du dépôt `gitops-demo` déjà sur GitHub.

Objectif d'entraînement : tout refaire sans regarder, en moins de 5 minutes.

---

## 0. Installer les outils

macOS :

```bash
brew install kubectl kind argocd
docker version   # Docker Desktop doit être lancé
```

Linux (amd64) :

```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl && rm kubectl

# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# argocd
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd /usr/local/bin/argocd && rm argocd
```

## 1. Récupérer le dépôt

```bash
git clone https://github.com/<VOTRE-USER>/gitops-demo.git
cd gitops-demo
```

## 2. Créer le cluster

```bash
cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: formation
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
EOF

kind create cluster --config kind-config.yaml
kubectl get nodes
```

## 3. Installer ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

## 4. Accéder à ArgoCD

Dans un **deuxième terminal**, à laisser ouvert :

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Puis dans le premier terminal :

```bash
argocd admin initial-password -n argocd
argocd login localhost:8080 --username admin --insecure
```

Interface web : https://localhost:8080 (utilisateur `admin`).

Dépôt privé uniquement :

```bash
argocd repo add https://github.com/<VOTRE-USER>/gitops-demo.git \
  --username <VOTRE-USER> --password <TOKEN>
```

## 5. Déployer hello

```bash
kubectl apply -f argocd/apps/hello-app.yaml

# ou, pour tout redéployer d'un coup (hello, dev, prod, podinfo) :
# kubectl apply -f argocd/root-app.yaml
```

Si le module 8 n'a pas été fait, le fichier est `argocd/hello-app.yaml`.

## 6. Vérifier

```bash
argocd app get hello --refresh
kubectl get pods -n hello
curl http://localhost:30080     # « Bonjour depuis Kubernetes ! »
```

## 7. Tout supprimer

```bash
kind delete cluster --name formation
```

---

## Rappel : les fichiers de l'app hello

Si le dépôt n'existe pas encore sur la nouvelle machine, voici les trois manifests à placer dans `apps/hello/` et l'Application ArgoCD.

`apps/hello/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-html
data:
  index.html: |
    <h1>Bonjour depuis Kubernetes !</h1>
    <p>Version 1</p>
```

`apps/hello/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  labels:
    app: hello
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 128Mi
          readinessProbe:
            httpGet:
              path: /
              port: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
      volumes:
        - name: html
          configMap:
            name: hello-html
```

`apps/hello/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  type: NodePort
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

`argocd/apps/hello-app.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<VOTRE-USER>/gitops-demo.git
    targetRevision: main
    path: apps/hello
  destination:
    server: https://kubernetes.default.svc
    namespace: hello
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```