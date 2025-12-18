# argocd-gitops-lab

Argo CD Hands-On Lab (iximiuz Kubernetes Playground)
Lab Objective

Install Argo CD

Deploy an application using pure GitOps

Practice sync, drift, self-heal, prune

Understand how Jenkins will fit later

Lab 1: Install Argo CD on iximiuz cluster
1. Verify cluster access
kubectl get nodes

2. Install Argo CD (official manifest)
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml


Wait for pods:

kubectl get pods -n argocd


You should see:

argocd-server

argocd-repo-server

argocd-application-controller

argocd-dex-server

Lab 2: Access Argo CD UI (iximiuz-friendly way)
Port-forward Argo CD server
kubectl port-forward svc/argocd-server -n argocd 8080:443


Open in browser:

https://localhost:8080


Ignore TLS warning (expected)

Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo


Login:

Username: admin

Password: (from above)

Lab 3: Create Git repo (VERY IMPORTANT STEP)

Create a new GitHub repo (public is easiest for labs), example:

argocd-gitops-lab

Recommended repo structure
argocd-gitops-lab/
├── apps/
│   └── demo-nginx/
│       ├── deployment.yaml
│       └── service.yaml
└── argocd/
    └── demo-nginx-app.yaml

Lab 4: Application manifests (copy exactly)
apps/demo-nginx/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo-nginx
  template:
    metadata:
      labels:
        app: demo-nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80

apps/demo-nginx/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: demo-nginx
spec:
  selector:
    app: demo-nginx
  ports:
  - port: 80
    targetPort: 80

Lab 5: Create Argo CD Application (THE CORE CONCEPT)
argocd/demo-nginx-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<YOUR_GITHUB_USERNAME>/argocd-gitops-lab.git
    targetRevision: main
    path: apps/demo-nginx
  destination:
    server: https://kubernetes.default.svc
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true


Apply it:

kubectl apply -f argocd/demo-nginx-app.yaml

Lab 6: Observe Argo CD in action (THIS IS WHERE YOU LEARN)
In Argo CD UI:

App status → Synced

Health → Healthy

Validate:

kubectl get all -n demo

Lab 7: Drift detection (mandatory real-world skill)
Manually break GitOps
kubectl scale deployment demo-nginx -n demo --replicas=5


What happens:

Argo CD shows OutOfSync

Argo CD automatically resets replicas back to 2

This is self-healing GitOps.

Lab 8: Pruning (deletion via Git)

Delete service.yaml from Git

Commit & push

Argo CD deletes Service automatically

This is why Argo CD is trusted as CD.

Lab 9: What Jenkins will do later (important mental model)

Your org flow will be:

Developer → Git Push
   ↓
Jenkins CI
   - Build image
   - Push to registry
   - Update image tag in Git (Helm/Kustomize)
   ↓
Argo CD
   - Detect Git change
   - Sync to cluster


Argo CD NEVER builds images.
It only reconciles Kubernetes state.

Lab 10: Next steps (I strongly recommend this order)

Rollback using Git commit history

Sync Waves (DB → App ordering)

App of Apps pattern

ApplicationSet

Helm + Kustomize with Argo CD

Multi-env (dev → stage → prod)

Next action for you

Reply with YES and tell me:

Do you want Helm or Kustomize next?

Or do you want Jenkins → Argo CD integration simulation?

I will give you the next lab accordingly, step by step.
