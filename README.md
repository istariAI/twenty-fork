
# Twenty – Local Development & Custom Container Deployment

This guide explains how to:

- Run Twenty (frontend + backend) **locally** for development.  
- **Build, push, and deploy** a custom backend image to your existing AKS cluster after changing UI or server code.

The Vars we need:

- Repo: `github.com/your-org/twenty` (forked)
- Registry: Docker Hub user `zeeshanhs`.  
- Azure Postgres server: `twentypsql.postgres.database.azure.com`, DB `twenty2`.  
- AKS already running a `twenty-server` Deployment.

Adjust names where needed.

***

## 1. Prerequisites

- Node.js **24.x** (use `nvm`).
- Yarn (classic or Berry).
- Docker Desktop (or any Docker engine).
- Access to:
  - Your Twenty fork.
  - Azure Postgres (`twentypsql` / `twenty2`).
  - AKS cluster (`kubectl` context configured).
  - Docker Hub account `zeeshanhs` (or your actual username).

***

## 2. Local development

### 2.1 Clone and install dependencies

```bash
# Clone your fork
git clone git@github.com:your-org/twenty.git
cd twenty

# Use Node 24
nvm use 24

# Install dependencies
yarn install
```

### 2.2 Configure environment

Copy example env files:

```bash
cp packages/twenty-front/.env.example  packages/twenty-front/.env
cp packages/twenty-server/.env.example packages/twenty-server/.env
```

Edit `packages/twenty-server/.env` and set the DB URL. For dev you can use the same Azure DB (just be aware you’re touching the “real” DB):

```env
PG_DATABASE_URL=postgresql://twentyadmin:YOUR_PASSWORD@twentypsql.postgres.database.azure.com:5432/twenty2?sslmode=require
```

Optionally also set:

```env
DATABASE_URL=${PG_DATABASE_URL}
```

### 2.3 Initialize the database (only if empty)

If `twenty2` is fresh and you need core tables and seed data:

```bash
# From repo root
export PG_DATABASE_URL='postgresql://twentyadmin:YOUR_PASSWORD@twentypsql.postgres.database.azure.com:5432/twenty2?sslmode=require'
export DATABASE_URL="$PG_DATABASE_URL"

# Reset + migrations
yarn nx run twenty-server:database:reset
yarn database:migrate:prod
```

This creates the `core` schema, workspace schemas, and all Twenty tables.

### 2.4 Run backend and frontend locally

In one terminal (backend):

```bash
cd twenty
nvm use 24

yarn nx run twenty-server:start
```

In another terminal (frontend):

```bash
cd twenty
nvm use 24

yarn nx run twenty-front:start
```

By default (check docs / env):

- Backend: `http://localhost:3000`  
- Frontend: `http://localhost:3001`

Use this setup while editing UI and server code; changes will hot‑reload.

***

## 3. Build and push a custom backend image

After you’re happy with your changes locally, build a new Docker image and push it to your registry.

### 3.1 Choose an image name

For Docker Hub user `zeeshanhs`, use:

```bash
export IMAGE_NAME=zeeshanhs/twenty-server:v0.1.0
```

- `zeeshanhs` – your Docker Hub username.  
- `twenty-server` – repo name in Docker Hub.  
- `v0.1.0` – any tag you like (`feature-x`, `2025-12-15`, etc.), all lowercase, no spaces.

### 3.2 Build the image from your fork

From the repo root:

```bash
cd twenty

docker build \
  -f packages/twenty-docker/Dockerfile \
  -t "$IMAGE_NAME" .
```

This Dockerfile builds the server (and usually bundles the frontend build) from your fork.

### 3.3 Log in and push

```bash
docker login           # enters Docker Hub credentials for user `zeeshanhs`
docker push "$IMAGE_NAME"
```

You should see the image appear at:  
`https://hub.docker.com/r/zeeshanhs/twenty-server/tags`

***

## 4. Deploy the new image to AKS

Assume you already have a Deployment file (for example `k8s/twenty-server-deploy.yaml`) that currently uses the official image.

### 4.1 Update the Deployment to use your image

Edit your Deployment YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: twenty-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: twenty-server
  template:
    metadata:
      labels:
        app: twenty-server
    spec:
      containers:
        - name: twenty-server
          image: zeeshanhs/twenty-server:v0.1.0   # ← your custom image
          env:
            - name: PG_DATABASE_URL
              value: "postgresql://twentyadmin:YOUR_PASSWORD@twentypsql.postgres.database.azure.com:5432/twenty2?sslmode=require"
            - name: DATABASE_URL
              value: "postgresql://twentyadmin:YOUR_PASSWORD@twentypsql.postgres.database.azure.com:5432/twenty2?sslmode=require"
            # …other environment variables (SERVER_URL, etc.)
```

### 4.2 Apply and roll the deployment

```bash
# Apply the updated manifest
kubectl apply -f k8s/twenty-server-deploy.yaml

# Restart pods to force pull the new image (optional but explicit)
kubectl delete pod -l app=twenty-server

# Watch status
kubectl get pods -l app=twenty-server
kubectl logs -l app=twenty-server --tail=80
```

Once the pod is `Running` and logs look healthy, your public endpoint  
(e.g. `http://20.50.189.151` or your custom domain) serves the new code.

***

## 5. Typical development → deploy loop

1. **Pull latest** from your fork’s main branch.  
2. **Create feature branch**: `git checkout -b feature/new-ui`.  
3. **Develop locally**:
   - `yarn nx run twenty-server:start`
   - `yarn nx run twenty-front:start`
4. **Build and push image**:
   ```bash
   export IMAGE_NAME=zeeshanhs/twenty-server:feature-new-ui-1
   docker build -f packages/twenty-docker/Dockerfile -t "$IMAGE_NAME" .
   docker push "$IMAGE_NAME"
   ```
5. **Update AKS Deployment** `image:` to this tag and apply.  
6. **Verify**:
   - `kubectl logs -l app=twenty-server --tail=80`
   - Open the app in browser and test the new UI / behavior.
