# 🚀 Node.js CI Pipeline — ECR + EKS + ArgoCD

A complete CI/CD pipeline for a Node.js application. When code is pushed to the main branch, GitHub Actions automatically builds, tests, and pushes the Docker image to Amazon ECR. ArgoCD then detects the change and deploys the updated image to Amazon EKS — fully automated, zero manual steps.

## 📌 Key Highlights

- Matrix CI build on Node.js 18 & 20 with lint and test stages
- Docker image tagged with commit SHA for full traceability
- Secure AWS ECR authentication via GitHub Actions secrets
- GitOps-based CD using ArgoCD — Git is the single source of truth
- ArgoCD auto-syncs on every manifest change with self-heal enabled
- NodePort service — no extra AWS Load Balancer cost

## 🗂️ Repository Structure

```
Node-App-CI-Pipeline-ECR/
├── .github/
│ └── workflows/
│     └── ci-ecr.yml # CI pipeline (test + docker + ECR)
├── public/
│ └── index.html # Static homepage
├── k8s/
|  └── manifests/
|      └── deployment.yaml
|      └── service.yaml
├── src/
│ └── app.js # Node.js server + /health endpoint
├── Dockerfile # Multi-stage Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
└── README.md

```

## 🏗️ Architecture

```
Code Push → GitHub Actions CI
               ├── Build & Test (Node 18, 20)
               ├── Build Docker Image
               ├── Push to ECR (commit SHA tag)
               └── Update deployment.yaml in CD repo
                          │
                          ▼
                   ArgoCD (this repo)
                   └── Detects Git change → Auto-syncs
                          │
                          ▼
                   EKS Cluster (ap-south-1)
                   └── Pulls image from ECR → Deploys pod
                          │
                          ▼
                   App live at http://<NODE-IP>:30007
```


## 🌐 Application Endpoints

| Endpoint | Description           |
|----------|-----------------------|
| /        | Serves static HTML page |
| /health  | Health check endpoint |


## ⚙️ Prerequisites
 
### Tools
 
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [eksctl](https://eksctl.io/installation/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- Git


### AWS IAM User
 
Create an IAM user (e.g. `EKSAdmin`) and attach the following managed policies:
- AWSCloudFormationFullAccess
- AmazonEC2FullAccess
- AmazonVPCFullAccess
- AmazonEKSClusterPolicy
- AmazonEKSWorkerNodePolicy
- IAMFullAccess
- AmazonEC2ContainerRegistryPowerUser

## 🚀 Setup & Deployment
Creates the VPC, subnets, security groups, EKS control plane, and EC2 node automatically. Takes ~15-20 minutes.
 
```bash
eksctl create cluster \
  --name node-app-cluster \
  --region ap-south-1 \
  --nodegroup-name node-app-ng \
  --node-type t3.medium \
  --nodes 1 \
  --managed
```
 
Verify the node is ready and note the `EXTERNAL-IP`:
 
```bash
kubectl get nodes -o wide
```
 
---
 
### Step 2 — Install ArgoCD
 
```bash
kubectl create namespace argocd
 
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml \
  --server-side
```
 
> The `--server-side` flag avoids an annotation size error on CRD resources.
 
Wait for all pods to be Running:
 
```bash
kubectl get pods -n argocd
```
 
**Expose the ArgoCD UI via NodePort:**
 
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc argocd-server -n argocd
```
 
The output shows something like `80:30548/TCP,443:32352/TCP`. Use the port mapped to **443** for HTTPS access.
 
Open that port in the EC2 Security Group:
**AWS Console → EC2 → Instances → click node → Security tab → Security Group → Inbound rules → Edit → Add rule → Custom TCP → ArgoCD port → Source `0.0.0.0/0`**

**Get the admin password:**
 
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```
 
**Login:** Open `https://<NODE-EXTERNAL-IP>:<ARGOCD-PORT>` → accept the certificate warning → login with `admin` and the password above.
 
---


### Step 3 — Open App Port in Security Group
 
The app runs on NodePort `30007`. Add it to the EC2 Security Group the same way as above — **Custom TCP → Port `30007` → Source `0.0.0.0/0`**.
 
---
 
### Step 4 — Add GitHub Secrets
 
In the **CI repo** → Settings → Secrets and variables → Actions → New repository secret:
 
| Secret | Value |
|--------|-------|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `AWS_REGION` | `ap-south-1` |
| `GH_TOKEN` | GitHub Personal Access Token (see below) |
 
**Create `GH_TOKEN`:**
GitHub → Profile → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token → select scopes `repo` and `workflow` → copy and save as the `GH_TOKEN` secret.
 
---
 
### Step 5 — Create ArgoCD Application
 
In the ArgoCD UI, click **+ New App** and fill in:
 
**General**
 
| Field | Value |
|-------|-------|
| Application Name | `node-app` |
| Project | `default` |
| Sync Policy | `Automatic` |
| Prune Resources | ✅ Enabled |
| Self Heal | ✅ Enabled |
 
**Source**
 
| Field | Value |
|-------|-------|
| Repo URL | `https://github.com/karan-karan/Node-App-CI-CD-Pipeline-ECR-EKS.git` |
| Revision | `main` |
| Path | `k8s/manifests` |
 
**Destination**
 
| Field | Value |
|-------|-------|
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

Click **Create**. ArgoCD will sync and deploy automatically. The app appears as **Synced** and **Healthy** in the UI.
 
---
 
### Step 6 — Verify
 
```bash
kubectl get pods
kubectl get svc
```
 
Access the app:
- `http://<NODE-EXTERNAL-IP>:30007/`
- `http://<NODE-EXTERNAL-IP>:30007/health`
---
 
## 🔄 CI/CD Flow (Automated)
 
Every code push to `main` triggers the full pipeline automatically:
 
| Step | What Happens |
|------|-------------|
| 1 | Code pushed to `main` in CI repo |
| 2 | GitHub Actions builds and tests on Node.js 18 and 20 |
| 3 | Docker image built and tagged with first 8 chars of commit SHA |
| 4 | Image pushed to Amazon ECR |
| 5 | `deployment.yaml` in this CD repo updated with new image tag |
| 6 | ArgoCD detects the Git change and syncs to EKS |
| 7 | New pod deployed — app live with latest code |
 
---
 
## 🔑 GitHub Actions Secrets Reference
 
| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | EKSAdmin IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | EKSAdmin IAM user secret key |
| `AWS_REGION` | AWS region — `ap-south-1` |
| `GH_TOKEN` | GitHub PAT with `repo` and `workflow` scopes |
 
---
 
## 🧹 Cleanup
Delete all resources to stop AWS charges:
 
```bash
kubectl delete namespace argocd
eksctl delete cluster --name node-app-cluster --region ap-south-1
eksctl get cluster --region ap-south-1
```


## 🐛 Common Errors & Fixes
 
**`AccessDeniedException` on eksctl**
Attach the `EKSFullAccess` inline policy to the IAM user. Wait 1-2 minutes for propagation, then retry.
 
**ArgoCD install — annotation too long**
Use the `--server-side` flag with the `kubectl apply` command.
 
**CI pipeline triggering in a loop**
The `paths-ignore` block in the CI workflow must be indented inside `push`. It ignores changes to `k8s/manifests/**` to prevent the CD repo commit from re-triggering CI.
 
**Permission denied pushing to CD repo**
Regenerate `GH_TOKEN` with `repo` and `workflow` scopes and update the secret in the CI repo.
 
**App showing old content after deploy**
Check that all 3 CI jobs passed. Verify the image tag in `deployment.yaml` is updated. Check ArgoCD shows **Synced**. Run `kubectl describe pod <pod-name> | grep Image` to confirm the running image. Hard refresh with `Ctrl+Shift+R`.
 
---



## 👨‍💻 Author

Karan
