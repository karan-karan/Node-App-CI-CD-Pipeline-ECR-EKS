# 🚀 Node.js CI Pipeline to AWS ECR (GitHub Actions)

A CI pipeline for a Node.js application that performs matrix builds, linting, testing, and then builds & pushes a Docker image to AWS Elastic Container Registry (ECR) using GitHub Actions.


## 📌 Key Highlights

- Static content served from public/
- /health endpoint for Docker, ECS, and ALB health checks
- Matrix CI (Linux & Windows, Node.js 18 & 20)
- Automated Lint + Test stage
- Multi-stage Docker build for optimized image size
- Secure AWS ECR authentication via GitHub Actions 

## 🗂️ Repository Structure

```
Node-App-CI-Pipeline-ECR/
├── .github/
│ └── workflows/
│ └── ci-ecr.yml # CI pipeline (test + docker + ECR)
├── public/
│ └── index.html # Static homepage
├── src/
│ └── app.js # Node.js server + /health endpoint
├── Dockerfile # Multi-stage Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
└── README.md

```

## 🌐 Application Endpoints

| Endpoint | Description           |
|----------|-----------------------|
| /        | Serves static HTML page |
| /health  | Health check endpoint |

## Usage

### ▶️ Run Locally (Node.js)

Prerequisites
- Node.js 18+

Steps
```bash
npm install
npm start
```
Open in browser:
```bash
http://localhost:3000
http://localhost:3000/health
```

### 🐳 Run Locally with Docker

Build Image
```bash
docker build -t node-app .
```
Run Container
```bash
docker run -p 3000:3000 node-app
```

## 🔄 CI Pipeline Overview (GitHub Actions)

Pipeline Stages
### 1️⃣ Build, Lint & Test (Matrix)
- OS: ubuntu-latest, windows-latest
- Node.js: 18, 20
- Steps:
  - Install dependencies
  - Run ESLint
  - Run tests

### 2️⃣ Docker Build & Push (ECR)
- Runs only if matrix tests succeed
- Builds Docker image
- Authenticates to AWS ECR
- Pushes image to ECR

## 🔐 AWS & GitHub Configuration
### AWS ECR (One-Time Setup)
- Create a private ECR repository:
**node-app**

### IAM Permissions
Attach one of the following to the IAM user used by GitHub Actions:
#### ✅ Recommended: **AmazonEC2ContainerRegistryPowerUser**

## 🔑 GitHub Secrets
Add these secrets in:

**GitHub → Settings → Secrets → Actions**
| Secret Name | Description           |
|----------|-----------------------|
| AWS_ACCESS_KEY_ID  | IAM access key |
| AWS_SECRET_ACCESS_KEY  | IAM secret key |
| AWS_REGION | AWS region (e.g. ap-south-1) |


## ECS / ALB Ready

The /health endpoint can be directly used for:
- ECS container health checks
- ALB target group health checks

## 👨‍💻 Author

Karan
