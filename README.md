# Eshtry-Mny — MERN Microservices + DevSecOps + GitOps (Jenkins, SonarQube, Trivy, Helm, Argo CD)


## Architecture diagram

![Eshtry-Mny architecture diagram](docs/architecture-diagram.png)


An e-commerce demo built as **MERN microservices** (Node.js/Express + MongoDB + React) and deployed to **Kubernetes** using **Helm**, with a **DevSecOps CI/CD pipeline** in **Jenkins** and **GitOps continuous delivery** via **Argo CD**.



## What’s in this repo

### Application services

- **Frontend**: React (Vite) in `front-end/` (dev server on `5173`)
- **User service**: Node.js/Express in `User/` (API on `9001`)
- **Product service**: Node.js/Express in `Product/` (API on `9000`)
- **Cart service**: Node.js/Express in `Cart/` (API on `9003`)
- **Database**: MongoDB (Atlas in this setup)


### DevSecOps & GitOps

- **CI/CD**: `Jenkinsfile`
  - Static code analysis with **SonarQube**
  - Dependency checks with **npm audit**
  - Container image scanning with **Trivy**
  - Build + push images to **Docker Hub**
  - Deploy to Kubernetes using **Helm**
- **Kubernetes**:
  - Raw manifests for reference in `k8s/base/`
  - Helm chart in `eshtry-mny/`
- **Argo CD**: GitOps application definition in `argocd-application.yaml`

## Repository structure 

```text
.
├─ Jenkinsfile
├─ docker-compose.yml
├─ argocd-application.yaml
├─ eshtry-mny/                 # Helm chart (templates + values)
├─ k8s/base/                   # Plain Kubernetes manifests (reference)
├─ front-end/
├─ User/
├─ Product/
└─ Cart/
```

## CI/CD pipeline (DevSecOps)

The Jenkins pipeline in `Jenkinsfile` implements these stages:

- **Checkout Code**: fetch repository source
- **SonarQube Analysis**: runs `sonar-scanner` against the codebase (project key: `eshtry-mny`)
- **Build Docker Images**: builds images for `User`, `Product`, `Cart`, and `front-end`
- **Security: Dependency Audit**: runs `npm audit` for each Node project (pipeline continues even if audit reports issues)
- **Security: Docker Scan (Trivy)**: scans each built image for **HIGH/CRITICAL** vulnerabilities (pipeline continues after reporting)
- **Push Images to Docker Hub**: authenticates using Jenkins credentials and pushes versioned images (tag = Jenkins `BUILD_NUMBER`)
- **Deploy to Kubernetes**: runs `helm upgrade --install` and overrides image tags with the build tag

## GitOps delivery with Argo CD

`argocd-application.yaml` defines an Argo CD `Application` named `eshtry-mny` that:

- Pulls the Helm chart from the repo path `eshtry-mny/`
- Deploys into the `default` namespace
- Uses **automated sync** with:
  - **prune**: removes deleted manifests
  - **selfHeal**: reconciles drift automatically

## Kubernetes + Helm deployment

### Helm chart

The Helm chart in `eshtry-mny/` templates:

- Deployments + Services for **user**, **product**, **cart**, and **frontend**
- A shared `ConfigMap` (`app-config`) for non-secret settings
- A shared `Secret` (`app-secrets`) for sensitive settings
- An `Ingress` that routes:
  - `/` → frontend
  - `/api/users` → user service
  - `/api/products` → product service
  - `/api/cart` → cart service

### Example install/upgrade

From the repo folder:

```bash
cd eshtry-mny
helm upgrade --install eshtry-mny . -n default
```

## Run locally (Docker Compose)

The `docker-compose.yml` runs all services locally:

```bash
docker compose up --build
```

Ports:

- `5173` → frontend
- `9001` → user service
- `9000` → product service
- `9003` → cart service

## Tooling screenshots

### MongoDB Atlas (documents)

![MongoDB Atlas users collection](docs/screenshots/screenshot-01.png)

### MongoDB Atlas (data model diagram)

![MongoDB Atlas data model](docs/screenshots/screenshot-02.png)

### Jenkins pipeline run (stages)

![Jenkins pipeline stages](docs/screenshots/screenshot-03.png)

### SonarQube project overview 

![SonarQube overview](docs/screenshots/screenshot-04.png)

### SonarQube issues list

![SonarQube issues](docs/screenshots/screenshot-05.png)

### SonarQube security hotspot review

![SonarQube security hotspots](docs/screenshots/screenshot-06.png)

### Argo CD application tree (synced/healthy)

![Argo CD application tree](docs/screenshots/screenshot-07.png)

### Kubernetes Dashboard (workloads overview)

![Kubernetes Dashboard workloads](docs/screenshots/screenshot-08.png)

### Kubernetes Dashboard (replica sets/services)

![Kubernetes Dashboard replica sets](docs/screenshots/screenshot-09.png)

## Security note (important)

This repo currently contains example secrets in plain text under `k8s/base/secret.yaml` and `eshtry-mny/values.yaml`.

For a real environment, replace these with one of:

- Kubernetes secrets created out-of-band (not committed)
- External secret managers (e.g., External Secrets Operator + Vault / cloud secret manager)
- Sealed Secrets / SOPS-encrypted values
