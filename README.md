Pathbased Routing (Kubernetes + Argo CD)

Multi-service demo that routes a single hostname to multiple static frontends by path:

/ → Landing page

/aws → AWS page

/azure → Azure page

/gcp → GCP page

Built to be production-style:

GitOps with Argo CD

Kustomize at repo root

Clean Ingress + SPA fallback in NGINX to avoid 404s

Architecture
[ Client ] ---> [ Ingress Controller (ALB or NGINX) ] ---> Services
                     |        |        |        |
                     |        |        |        +--> /gcp  -> web-gcp (nginx)
                     |        |        +----------> /azure-> web-azure (nginx)
                     |        +--------------------> /aws  -> web-aws (nginx)
                     +------------------------------> /     -> web-landing (nginx)


Each service is a small NGINX container that serves index.html and uses a fallback:

location / { try_files $uri $uri/ /index.html; }


This ensures deep links like /aws/foo don’t 404.

Repo Layout
Pathbased-routing/
├─ aws/
│  ├─ Dockerfile
│  ├─ index.html
│  ├─ nginx.conf
│  ├─ deploy.yaml
│  └─ svc.yaml
├─ azure/
│  ├─ Dockerfile
│  ├─ index.html
│  ├─ nginx.conf
│  ├─ deploy.yaml
│  └─ svc.yaml
├─ gcp/
│  ├─ Dockerfile
│  ├─ index.html
│  ├─ nginx.conf
│  ├─ deploy.yaml
│  └─ svc.yaml
├─ deploy.yaml          # landing page deployment (web-landing)
├─ svc.yaml             # landing page service
├─ ingress.yaml         # routes /, /aws, /azure, /gcp
├─ kustomization.yaml   # aggregates all the above
└─ README.md


kustomization.yaml (root):

resources:
  - aws/deploy.yaml
  - aws/svc.yaml
  - azure/deploy.yaml
  - azure/svc.yaml
  - gcp/deploy.yaml
  - gcp/svc.yaml
  - deploy.yaml
  - svc.yaml
  - ingress.yaml

# Optional: centralize image tags here (see “Update images” section)
# images:
#   - name: docker.io/harish0604/pathrouter-aws
#     newTag: v4
#   - name: docker.io/harish0604/pathrouter-azure
#     newTag: v4
#   - name: docker.io/harish0604/pathrouter-gcp
#     newTag: v4
#   - name: docker.io/harish0604/pathrouter-landing
#     newTag: v2

Prerequisites

A Kubernetes cluster (EKS, AKS, GKE, minikube, Kind, etc.)

Ingress controller:

For AWS EKS: AWS Load Balancer Controller (ALB) or NGINX Ingress Controller

Argo CD installed in the cluster:

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

Build & Push Images

Each service has its own Dockerfile and optional nginx.conf:

# AWS page
docker build -t docker.io/<your-dockerhub>/pathrouter-aws:v1 ./aws
docker push docker.io/<your-dockerhub>/pathrouter-aws:v1

# Azure page
docker build -t docker.io/<your-dockerhub>/pathrouter-azure:v1 ./azure
docker push docker.io/<your-dockerhub>/pathrouter-azure:v1

# GCP page
docker build -t docker.io/<your-dockerhub>/pathrouter-gcp:v1 ./gcp
docker push docker.io/<your-dockerhub>/pathrouter-gcp:v1

# Landing page
docker build -t docker.io/<your-dockerhub>/pathrouter-landing:v1 .
docker push docker.io/<your-dockerhub>/pathrouter-landing:v1


Update the deployment YAMLs to point at your images (or use Kustomize images:—see below).

Argo CD Application
Option A: Create via YAML

app-argocd.yaml:

apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: pathbased-routing
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-gh>/Pathbased-routing.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: pathrouter
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true


Apply:

kubectl apply -f app-argocd.yaml

Option B: Create in Argo CD UI

New App →
Repo URL: your GitHub repo
Revision: main
Path: .
Destination: in-cluster, Namespace: pathrouter
Sync Policy: Auto-Sync + Prune + Self Heal
Sync Options: Auto-Create Namespace

Ingress (Path Routing)

ingress.yaml (works for ALB or NGINX—no regex, no rewrites):

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-router
  namespace: pathrouter
  annotations:
    kubernetes.io/ingress.class: nginx  # or: alb (for AWS LB Controller)
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: web-landing, port: { number: 80 } } }
      - path: /aws
        pathType: Prefix
        backend: { service: { name: web-aws, port: { number: 80 } } }
      - path: /azure
        pathType: Prefix
        backend: { service: { name: web-azure, port: { number: 80 } } }
      - path: /gcp
        pathType: Prefix
        backend: { service: { name: web-gcp, port: { number: 80 } } }


Because we don’t rewrite paths, each container should include try_files ... /index.html to prevent 404s on deep links.

Verify
kubectl -n pathrouter get deploy,svc,ingress
kubectl -n pathrouter get pods -o wide
kubectl -n pathrouter get ingress path-router -o wide


Get the Ingress address/DNS (depends on controller), then open:

http://<ingress>/

http://<ingress>/aws/

http://<ingress>/azure/

http://<ingress>/gcp/

Update Images (GitOps)

Best practice: change images in Git, not with kubectl set image (Argo will revert manual changes).

Option 1: Edit Deployment YAMLs

Update the image: field in aws/deploy.yaml, azure/deploy.yaml, gcp/deploy.yaml, and landing deploy.yaml.

Option 2: Use Kustomize images: (central control)

Add to root kustomization.yaml:

images:
  - name: docker.io/<your-dockerhub>/pathrouter-aws
    newTag: v4
  - name: docker.io/<your-dockerhub>/pathrouter-azure
    newTag: v4
  - name: docker.io/<your-dockerhub>/pathrouter-gcp
    newTag: v4
  - name: docker.io/<your-dockerhub>/pathrouter-landing
    newTag: v2


Ensure the base image names in the deployment YAMLs match the name: entries (without tags).

Commit & push → Argo syncs → rolling update.

Rollback

Argo CD UI → Application → History → select a previous Sync → Rollback.

Or revert the Git commit that bumped the image tag.

Troubleshooting
404 on /azure or /gcp

Service endpoints empty → label mismatch. Ensure:

Deployment.spec.selector.matchLabels == Pod template labels == Service.spec.selector

Containers 404 deep links → ensure the NGINX fallback exists:

location / { try_files $uri $uri/ /index.html; }


Links: your landing page buttons should point to /aws/, /azure/, /gcp/ (with slash).

Ingress rejects regex paths

Use simple pathType: Prefix and let containers handle SPA fallback. (K8s API requires “absolute paths”; regex won’t validate in path fields.)

NodePort not reachable from laptop (EKS)

NodePort opens on node private IPs. Use:

Ingress/LoadBalancer (recommended), or

kubectl port-forward svc/web-aws 8080:80 → http://localhost:8080

“Immutable Service” sync errors

Add to each Service:

metadata:
  annotations:
    argocd.argoproj.io/sync-options: Replace=true

Argo keeps undoing my changes

That’s expected—Git is the source of truth. Edit YAMLs in Git (or Kustomize images) and push.

TLS (Optional)

NGINX Ingress + cert-manager:

Install cert-manager, create a ClusterIssuer, then add:

metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  tls:
  - hosts: [your.domain.com]
    secretName: tls-your-domain


AWS ALB + ACM:

Use kubernetes.io/ingress.class: alb and set the ACM ARN annotations on the Ingress; ALB will terminate TLS.

Contributing

Fork / branch

Update HTML or Dockerfile/nginx.conf per service

Build & push image

Bump image tag via Deployment YAML or kustomization.yaml

Commit & PR

License

MIT (or your preferred license)

Notes

Container names should match their app (web-aws, web-azure, web-gcp, web-landing) for clarity.

Prefer immutable image tags (v1, v2 …) over latest.

Consider Argo CD Image Updater later to auto-bump tags in Git.
