# E-Commerce MERN Microservices

## Description
E-commerce web app (games store) built with the MERN stack using a microservices architecture.

## Services
- **Frontend**: `front-end` (Vite/React) — port `5173`
- **Product service**: `Product` — port `9000`
- **User service**: `User` — port `9001`
- **Cart service**: `Cart` — port `9003`

## Prerequisites
- Node.js + npm (for local development)
- Docker + Docker Compose (recommended for local development)
- Kubernetes + `kubectl` (for cluster deployment)

## Environment variables
This repo uses env vars both for local Docker Compose and Kubernetes.

- **Secrets (do not commit real values)**:
  - `MONGO_USERNAME`
  - `MONGO_PASSWORD`
  - `ACCESS_TOKEN`
- **Non-secret config**:
  - `MONGO_CLUSTER`
  - `MONGO_DBNAME`
  - `NODE_ENV`

### Local `.env`
Create a `.env` file in the repo root (used by `docker-compose.yml`):

```bash
MONGO_USERNAME=your_mongo_user
MONGO_PASSWORD=your_mongo_password
ACCESS_TOKEN=your_access_token

MONGO_CLUSTER=cluster0.yourcluster.mongodb.net
MONGO_DBNAME=your_db_name
NODE_ENV=development
```

Each service may also have its own env file (Compose references `./User/.env` and `./Cart/.env` if present).

## Run locally with Docker Compose
From the repo root:

```bash
docker compose up --build
```

Then open:
- **Frontend**: `http://localhost:5173/`

Backend ports are exposed for local testing:
- **Product**: `http://localhost:9000/`
- **User**: `http://localhost:9001/`
- **Cart**: `http://localhost:9003/`

## Deploy to Kubernetes
Manifests live under `k8s/base/`:
- `configmap.yaml` (`app-config`)
- `secret.yaml` (`app-secrets`)
- `services.yaml` (cluster services + frontend NodePort)
- `*/deployment.yaml` for each service

### Important security note (Secrets)
`k8s/base/secret.yaml` currently contains example values. Replace them before deploying, or create the secret dynamically:

```bash
kubectl create secret generic app-secrets \
  --from-literal=MONGO_USERNAME=your_mongo_user \
  --from-literal=MONGO_PASSWORD=your_mongo_password \
  --from-literal=ACCESS_TOKEN=your_access_token
```

### Apply manifests

```bash
kubectl apply -f k8s/base/configmap.yaml
kubectl apply -f k8s/base/secret.yaml
kubectl apply -f k8s/base/services.yaml
kubectl apply -f k8s/base/user/deployment.yaml
kubectl apply -f k8s/base/product/deployment.yaml
kubectl apply -f k8s/base/cart/deployment.yaml
kubectl apply -f k8s/base/frontend/deployment.yaml
```

### Access the frontend (NodePort)
The `frontend` Kubernetes `Service` is `NodePort`. Find the assigned port and open it on any node IP:

```bash
kubectl get svc frontend
```

## Data
- `products.json` contains example products for the app.

## Tech stack
- React (Vite)
- Node.js / Express
- MongoDB
- Docker / Docker Compose
- Kubernetes
