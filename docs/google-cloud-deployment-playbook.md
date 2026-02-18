# Google Cloud Deployment Playbook

## Deploy OpenClaw on GCP: From Zero to Production

This playbook provides comprehensive, step-by-step instructions for deploying OpenClaw -- a multi-channel AI gateway with extensible messaging integrations -- on Google Cloud Platform. It covers three deployment models (Cloud Run, GKE, and Compute Engine), plus supporting infrastructure for databases, storage, networking, security, monitoring, CI/CD, and cost optimization.

All commands and configurations reference the actual OpenClaw codebase. OpenClaw version at time of writing: `2026.2.16`.

---

## Prerequisites

### GCP Account and Tooling

1. **GCP Account** with billing enabled. Navigate to [https://console.cloud.google.com/billing](https://console.cloud.google.com/billing) to set up a billing account.

2. **gcloud CLI** installed and authenticated:

   ```bash
   # Install: https://cloud.google.com/sdk/docs/install
   gcloud init
   gcloud auth login
   gcloud auth configure-docker
   ```

3. **Enable required GCP APIs** (run once per project):

   ```bash
   gcloud services enable \
     compute.googleapis.com \
     run.googleapis.com \
     artifactregistry.googleapis.com \
     container.googleapis.com \
     secretmanager.googleapis.com \
     cloudbuild.googleapis.com \
     monitoring.googleapis.com \
     logging.googleapis.com \
     sqladmin.googleapis.com
   ```

4. **Set your project and region defaults**:

   ```bash
   export GCP_PROJECT="my-openclaw-project"
   export GCP_REGION="us-central1"
   export GCP_ZONE="us-central1-a"
   gcloud config set project "$GCP_PROJECT"
   gcloud config set compute/region "$GCP_REGION"
   gcloud config set compute/zone "$GCP_ZONE"
   ```

### Local Tools Required

- **Node.js 22+** (OpenClaw requires `>=22.12.0` per `package.json` `engines` field)
- **pnpm 10+** (package manager; `packageManager: "pnpm@10.23.0"`)
- **Docker** (for building container images)
- **Bun** (optional; used by build scripts, installed in the Dockerfile via `curl -fsSL https://bun.sh/install | bash`)

### Clone the Repository

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
```

---

## Part 1: Cloud Run Deployment (Recommended)

Cloud Run is the recommended GCP deployment target for OpenClaw. It provides managed container hosting with automatic HTTPS, scaling, and zero server management. This maps closely to the existing Fly.io and Render deployments already defined in `fly.toml` and `render.yaml`.

### 1.1 Create an Artifact Registry Repository

```bash
gcloud artifacts repositories create openclaw-repo \
  --repository-format=docker \
  --location="$GCP_REGION" \
  --description="OpenClaw container images"
```

### 1.2 Build and Push the Container Image

The Dockerfile builds a production image based on `node:22-bookworm`. Key characteristics:

- Installs Bun (required for build scripts)
- Enables corepack for pnpm
- Supports optional `OPENCLAW_DOCKER_APT_PACKAGES` build arg for extra system packages
- Runs `pnpm install --frozen-lockfile`, `pnpm build`, and `pnpm ui:build`
- Sets `NODE_ENV=production`
- Runs as the non-root `node` user (uid 1000) for security
- Default CMD: `node openclaw.mjs gateway --allow-unconfigured`

Build and push:

```bash
# Tag format: REGION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG
export IMAGE_URI="$GCP_REGION-docker.pkg.dev/$GCP_PROJECT/openclaw-repo/openclaw:latest"

# Authenticate Docker with Artifact Registry
gcloud auth configure-docker "$GCP_REGION-docker.pkg.dev"

# Build (add --build-arg for optional apt packages)
docker build \
  --build-arg OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg" \
  -t "$IMAGE_URI" \
  -f Dockerfile .

# Push
docker push "$IMAGE_URI"
```

Alternatively, use Cloud Build for remote builds:

```bash
gcloud builds submit \
  --tag "$IMAGE_URI" \
  --timeout=1800s
```

### 1.3 Store Secrets in Secret Manager

The gateway requires a token for non-loopback binds and supports various provider API keys:

```bash
# Generate and store gateway token
GATEWAY_TOKEN=$(openssl rand -hex 32)
echo -n "$GATEWAY_TOKEN" | gcloud secrets create openclaw-gateway-token --data-file=-

# Store model provider keys
echo -n "sk-ant-your-key" | gcloud secrets create anthropic-api-key --data-file=-
echo -n "sk-your-key" | gcloud secrets create openai-api-key --data-file=-

# Store channel tokens (as needed)
echo -n "your-discord-bot-token" | gcloud secrets create discord-bot-token --data-file=-
```

### 1.4 Deploy to Cloud Run

OpenClaw's gateway runs as a long-lived WebSocket server on a single port (default 18789). Cloud Run supports WebSocket connections and HTTP/2:

```bash
gcloud run deploy openclaw \
  --image="$IMAGE_URI" \
  --region="$GCP_REGION" \
  --platform=managed \
  --port=8080 \
  --memory=2Gi \
  --cpu=2 \
  --min-instances=1 \
  --max-instances=3 \
  --timeout=3600 \
  --session-affinity \
  --no-cpu-throttling \
  --execution-environment=gen2 \
  --set-env-vars="NODE_ENV=production" \
  --set-env-vars="OPENCLAW_PREFER_PNPM=1" \
  --set-env-vars="NODE_OPTIONS=--max-old-space-size=1536" \
  --set-env-vars="OPENCLAW_GATEWAY_PORT=8080" \
  --set-secrets="OPENCLAW_GATEWAY_TOKEN=openclaw-gateway-token:latest" \
  --set-secrets="ANTHROPIC_API_KEY=anthropic-api-key:latest" \
  --set-secrets="OPENAI_API_KEY=openai-api-key:latest" \
  --command="node" \
  --args="openclaw.mjs,gateway,--allow-unconfigured,--port,8080,--bind,lan" \
  --allow-unauthenticated
```

Key flags explained:

| Flag | Purpose |
|------|---------|
| `--min-instances=1` | Keeps at least one instance warm (like `auto_stop_machines = false` in fly.toml) |
| `--session-affinity` | Routes WebSocket connections to the same instance |
| `--no-cpu-throttling` | CPU stays allocated even between requests (critical for long-running gateway) |
| `--timeout=3600` | Maximum request duration of 1 hour for WebSocket connections |
| `--execution-environment=gen2` | Full Linux compatibility for Node.js native modules |
| `--memory=2Gi` | 2GB RAM recommended (512MB is too small; see fly.toml `memory = "2048mb"`) |

### 1.5 Persistent State with Cloud Storage FUSE

Cloud Run is stateless by default. OpenClaw stores state in `~/.openclaw/`. For persistence:

```bash
# Create a bucket for OpenClaw state
gsutil mb -l "$GCP_REGION" "gs://$GCP_PROJECT-openclaw-state"

# Grant the Cloud Run service account access
gcloud run services update openclaw \
  --region="$GCP_REGION" \
  --add-volume=name=openclaw-data,type=cloud-storage,bucket="$GCP_PROJECT-openclaw-state" \
  --add-volume-mount=volume=openclaw-data,mount-path=/data \
  --set-env-vars="OPENCLAW_STATE_DIR=/data"
```

This mirrors the persistent volume approach used in `fly.toml` (`OPENCLAW_STATE_DIR = "/data"`) and `render.yaml` (`OPENCLAW_STATE_DIR: /data/.openclaw`).

### 1.6 Custom Domain and SSL

Cloud Run provides automatic HTTPS on `*.run.app` domains. For a custom domain:

```bash
gcloud run domain-mappings create \
  --service=openclaw \
  --domain=openclaw.yourdomain.com \
  --region="$GCP_REGION"
```

Cloud Run handles TLS certificate provisioning and renewal automatically.

### 1.7 Auto-scaling Configuration

```bash
gcloud run services update openclaw \
  --region="$GCP_REGION" \
  --min-instances=1 \
  --max-instances=5 \
  --concurrency=80
```

For OpenClaw, vertical scaling (more memory/CPU per instance) is generally more effective than horizontal scaling, because the gateway maintains WebSocket state in-process.

---

## Part 2: GKE (Kubernetes) Deployment

For teams that need fine-grained orchestration, multi-service deployments, or already use Kubernetes.

### 2.1 Create a GKE Cluster

```bash
gcloud container clusters create openclaw-cluster \
  --zone="$GCP_ZONE" \
  --num-nodes=2 \
  --machine-type=e2-standard-2 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=5 \
  --disk-size=50

gcloud container clusters get-credentials openclaw-cluster --zone="$GCP_ZONE"
```

### 2.2 Kubernetes Manifests

#### Namespace and ConfigMap

```yaml
# openclaw-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: openclaw
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: openclaw-config
  namespace: openclaw
data:
  NODE_ENV: "production"
  OPENCLAW_PREFER_PNPM: "1"
  OPENCLAW_STATE_DIR: "/data"
  OPENCLAW_GATEWAY_PORT: "8080"
  NODE_OPTIONS: "--max-old-space-size=1536"
```

#### Deployment

```yaml
# openclaw-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: openclaw-gateway
  namespace: openclaw
  labels:
    app: openclaw
spec:
  replicas: 1
  selector:
    matchLabels:
      app: openclaw
  template:
    metadata:
      labels:
        app: openclaw
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
        - name: openclaw
          image: us-central1-docker.pkg.dev/PROJECT/openclaw-repo/openclaw:latest
          command: ["node", "openclaw.mjs", "gateway", "--allow-unconfigured", "--port", "8080", "--bind", "lan"]
          ports:
            - containerPort: 8080
              protocol: TCP
          envFrom:
            - configMapRef:
                name: openclaw-config
            - secretRef:
                name: openclaw-secrets
          resources:
            requests:
              memory: "1Gi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "2000m"
          volumeMounts:
            - name: openclaw-data
              mountPath: /data
          readinessProbe:
            exec:
              command: ["node", "openclaw.mjs", "health", "--token-from-env"]
            initialDelaySeconds: 15
            periodSeconds: 30
            timeoutSeconds: 10
          livenessProbe:
            exec:
              command: ["node", "openclaw.mjs", "health", "--token-from-env"]
            initialDelaySeconds: 30
            periodSeconds: 60
            timeoutSeconds: 10
      volumes:
        - name: openclaw-data
          persistentVolumeClaim:
            claimName: openclaw-pvc
```

#### PersistentVolumeClaim

```yaml
# openclaw-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: openclaw-pvc
  namespace: openclaw
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard-rwo
```

#### Service and Ingress

```yaml
# openclaw-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: openclaw-service
  namespace: openclaw
spec:
  selector:
    app: openclaw
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: openclaw-ingress
  namespace: openclaw
  annotations:
    kubernetes.io/ingress.class: "gce"
    networking.gke.io/managed-certificates: openclaw-cert
    kubernetes.io/ingress.global-static-ip-name: openclaw-ip
spec:
  rules:
    - host: openclaw.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: openclaw-service
                port:
                  number: 80
```

#### Horizontal Pod Autoscaler

```yaml
# openclaw-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: openclaw-hpa
  namespace: openclaw
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: openclaw-gateway
  minReplicas: 1
  maxReplicas: 3
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### 2.3 Deploy to GKE

```bash
kubectl apply -f openclaw-namespace.yaml
kubectl apply -f openclaw-configmap.yaml
kubectl apply -f openclaw-secrets.yaml
kubectl apply -f openclaw-pvc.yaml
kubectl apply -f openclaw-deployment.yaml
kubectl apply -f openclaw-service.yaml
kubectl apply -f openclaw-ingress.yaml
kubectl apply -f openclaw-hpa.yaml
```

---

## Part 3: GCE (VM) Deployment

Use this for maximum control, SSH-based access, and the simplest mental model.

### 3.1 Create the VM

```bash
gcloud compute instances create openclaw-gateway \
  --zone="$GCP_ZONE" \
  --machine-type=e2-small \
  --boot-disk-size=20GB \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=openclaw-gateway
```

Machine type recommendations:

| Type | Specs | Cost | Notes |
|------|-------|------|-------|
| e2-micro | 2 vCPU (shared), 1GB RAM | Free tier eligible | May OOM under load |
| e2-small | 2 vCPU, 2GB RAM | ~$12/mo | Recommended |
| e2-medium | 2 vCPU, 4GB RAM | ~$24/mo | For heavy workloads |

### 3.2 Install Docker and Deploy

```bash
gcloud compute ssh openclaw-gateway --zone="$GCP_ZONE"

# Install Docker
sudo apt-get update && sudo apt-get install -y git curl ca-certificates
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
exit

# Re-SSH for group change
gcloud compute ssh openclaw-gateway --zone="$GCP_ZONE"

# Clone and setup
git clone https://github.com/openclaw/openclaw.git
cd openclaw
mkdir -p ~/.openclaw ~/.openclaw/workspace

# Run the setup script
./docker-setup.sh
```

### 3.3 Systemd Service (Alternative to Docker)

```ini
# /etc/systemd/system/openclaw-gateway.service
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
User=openclaw
WorkingDirectory=/home/openclaw/openclaw
Environment=NODE_ENV=production
Environment=OPENCLAW_GATEWAY_TOKEN=your-token-here
Environment=OPENCLAW_GATEWAY_PORT=18789
ExecStart=/usr/bin/node openclaw.mjs gateway --bind lan --port 18789
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### 3.4 Nginx Reverse Proxy with SSL

```nginx
upstream openclaw_backend {
    server 127.0.0.1:18789;
}

server {
    listen 80;
    server_name openclaw.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name openclaw.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/openclaw.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/openclaw.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://openclaw_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

### 3.5 SSH Tunnel Access (Recommended for Security)

```bash
gcloud compute ssh openclaw-gateway --zone=us-central1-a -- -L 18789:127.0.0.1:18789
```

Then open `http://127.0.0.1:18789/` in your browser and paste the gateway token.

---

## Part 4: Database and Storage

### 4.1 OpenClaw State Persistence

| Component | Container Path | Persistence | Notes |
|-----------|----------------|-------------|-------|
| Gateway config | `/home/node/.openclaw/` | Host volume mount | Includes `openclaw.json`, tokens |
| Model auth profiles | `/home/node/.openclaw/` | Host volume mount | OAuth tokens, API keys |
| Skill configs | `/home/node/.openclaw/skills/` | Host volume mount | Skill-level state |
| Agent workspace | `/home/node/.openclaw/workspace/` | Host volume mount | Code and agent artifacts |
| WhatsApp session | `/home/node/.openclaw/` | Host volume mount | Preserves QR login |
| External binaries | `/usr/local/bin/` | Docker image | Must be baked at build time |

### 4.2 Cloud Storage for Backups

```bash
gsutil mb -l "$GCP_REGION" "gs://$GCP_PROJECT-openclaw-backups"
gsutil -m rsync -r /data/ "gs://$GCP_PROJECT-openclaw-backups/$(date +%Y-%m-%d)/"
```

### 4.3 Memorystore (Redis) for Caching

```bash
gcloud redis instances create openclaw-cache \
  --size=1 \
  --region="$GCP_REGION" \
  --redis-version=redis_7_0 \
  --tier=basic
```

---

## Part 5: Networking and Security

### 5.1 VPC and Firewall Rules

```bash
gcloud compute firewall-rules create openclaw-allow-ssh \
  --allow=tcp:22 --target-tags=openclaw-gateway --source-ranges="0.0.0.0/0"

gcloud compute firewall-rules create openclaw-allow-gateway \
  --allow=tcp:18789 --target-tags=openclaw-gateway --source-ranges="YOUR_IP/32"

gcloud compute firewall-rules create openclaw-allow-https \
  --allow=tcp:443 --target-tags=openclaw-gateway --source-ranges="0.0.0.0/0"
```

### 5.2 Cloud Armor (WAF)

```bash
gcloud compute security-policies create openclaw-waf \
  --description="OpenClaw WAF policy"

gcloud compute security-policies rules create 1000 \
  --security-policy=openclaw-waf \
  --action=throttle \
  --rate-limit-threshold-count=100 \
  --rate-limit-threshold-interval-sec=60 \
  --conform-action=allow \
  --exceed-action=deny-429 \
  --enforce-on-key=IP
```

### 5.3 IAM Roles and Service Accounts

```bash
gcloud iam service-accounts create openclaw-deploy \
  --display-name="OpenClaw Deployment"

gcloud projects add-iam-policy-binding "$GCP_PROJECT" \
  --member="serviceAccount:openclaw-deploy@$GCP_PROJECT.iam.gserviceaccount.com" \
  --role="roles/run.developer"

gcloud projects add-iam-policy-binding "$GCP_PROJECT" \
  --member="serviceAccount:openclaw-deploy@$GCP_PROJECT.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding "$GCP_PROJECT" \
  --member="serviceAccount:openclaw-deploy@$GCP_PROJECT.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

### 5.4 Cloud NAT for Outbound Traffic

```bash
gcloud compute routers create openclaw-router \
  --network=default --region="$GCP_REGION"

gcloud compute routers nats create openclaw-nat \
  --router=openclaw-router --region="$GCP_REGION" \
  --nat-all-subnet-ip-ranges --auto-allocate-nat-external-ips
```

---

## Part 6: Monitoring and Operations

### 6.1 Cloud Logging

```bash
gcloud logging metrics create openclaw-errors \
  --description="OpenClaw gateway errors" \
  --filter='resource.type="cloud_run_revision" AND resource.labels.service_name="openclaw" AND severity>=ERROR'
```

### 6.2 Alerting

```bash
gcloud alpha monitoring policies create \
  --display-name="OpenClaw Memory Pressure" \
  --condition-display-name="Memory > 85%" \
  --condition-filter='resource.type="cloud_run_revision" AND metric.type="run.googleapis.com/container/memory/utilizations"' \
  --condition-threshold-value=0.85 \
  --condition-threshold-comparison=COMPARISON_GT \
  --duration=60s
```

### 6.3 Gateway Health Verification

```bash
node openclaw.mjs health --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw gateway status --deep
openclaw channels status --probe
openclaw doctor
```

---

## Part 7: CI/CD Pipeline

### 7.1 Cloud Build

```yaml
# cloudbuild.yaml
steps:
  - name: 'node:22'
    entrypoint: 'bash'
    args: ['-c', 'corepack enable && pnpm install --frozen-lockfile && pnpm build && pnpm test:fast']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', '$_IMAGE_URI', '-f', 'Dockerfile', '.']
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', '$_IMAGE_URI']
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: 'gcloud'
    args: ['run', 'deploy', 'openclaw', '--image=$_IMAGE_URI', '--region=$_GCP_REGION', '--platform=managed']

substitutions:
  _GCP_REGION: us-central1
  _IMAGE_URI: us-central1-docker.pkg.dev/${PROJECT_ID}/openclaw-repo/openclaw:${SHORT_SHA}
```

### 7.2 Blue/Green Deployments

```bash
gcloud run deploy openclaw --image="$IMAGE_URI" --region="$GCP_REGION" --no-traffic --tag=canary
gcloud run services update-traffic openclaw --region="$GCP_REGION" --to-tags=canary=10
gcloud run services update-traffic openclaw --region="$GCP_REGION" --to-latest
```

### 7.3 Rollback

```bash
gcloud run revisions list --service=openclaw --region="$GCP_REGION"
gcloud run services update-traffic openclaw --region="$GCP_REGION" --to-revisions=openclaw-PREVIOUS_REVISION=100
```

---

## Part 8: Cost Optimization

| Deployment | Recommended Start | Monthly Estimate |
|------------|-------------------|------------------|
| Cloud Run (min 1, 2GB) | 1 vCPU, 2GB | ~$15-25/mo |
| GKE (1 node, e2-standard-2) | 2 vCPU, 8GB | ~$50-70/mo |
| GCE VM (e2-small) | 2 vCPU, 2GB | ~$12/mo |

### Tips

- Set `--min-instances=0` for dev (allows scale to zero)
- Use committed use discounts for 24/7 VMs (up to 57% savings)
- Use Spot instances for non-critical environments
- Set budget alerts: `gcloud billing budgets create --budget-amount=50USD`

---

## Appendix A: Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `OPENCLAW_GATEWAY_TOKEN` | Yes (non-loopback) | Auto-generated | Authentication token for gateway access |
| `OPENCLAW_GATEWAY_PASSWORD` | No | None | Alternative to token-based auth |
| `OPENCLAW_GATEWAY_PORT` | No | `18789` | Port the gateway listens on |
| `OPENCLAW_GATEWAY_BIND` | No | `loopback` | Bind mode: `loopback`, `lan`, `tailnet`, `auto` |
| `OPENCLAW_STATE_DIR` | No | `~/.openclaw` | Base directory for all persistent state |
| `OPENCLAW_CONFIG_DIR` | No | `~/.openclaw` | Directory containing `openclaw.json` config |
| `OPENCLAW_WORKSPACE_DIR` | No | `~/.openclaw/workspace` | Agent workspace directory |
| `OPENCLAW_PREFER_PNPM` | No | `0` | Force pnpm for UI build (ARM/Synology) |
| `OPENCLAW_DOCKER_APT_PACKAGES` | No | None | Extra apt packages for Docker build |
| `NODE_ENV` | No | `development` | Set to `production` for deployments |
| `NODE_OPTIONS` | No | None | Recommended: `--max-old-space-size=1536` |
| `ANTHROPIC_API_KEY` | No | None | Anthropic API key for Claude models |
| `OPENAI_API_KEY` | No | None | OpenAI API key |
| `GOOGLE_API_KEY` | No | None | Google AI API key |
| `GROQ_API_KEY` | No | None | Groq API key |
| `DISCORD_BOT_TOKEN` | No | None | Discord bot token |
| `CLAUDE_AI_SESSION_KEY` | No | None | Claude AI session key |

## Appendix B: Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Container won't start | Invalid config | Run `openclaw doctor --fix` |
| "Refusing to bind without auth" | Missing token | Set `OPENCLAW_GATEWAY_TOKEN` |
| OOM crashes | < 2GB RAM | Increase to `--memory=2Gi` |
| WebSocket drops | Timeout too short | Set `--timeout=3600` and `--session-affinity` |
| Permission errors | Wrong UID | `chown -R 1000:1000` on mounted volumes |
| State not persisting | No volume mount | Set `OPENCLAW_STATE_DIR=/data` with mounted volume |
