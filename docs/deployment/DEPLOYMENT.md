# Deployment & Serving Documentation — InfraNova AI

**Document Version:** 4.0  
**Date:** 2026-09-16  
**Status:** Production deployment runbook

## 1. Scope

This runbook deploys the existing **Exp9** production system. It does not change model architecture, retrain the model, or publish model weights.

Production system:

```text
Landsat 9 B10 + B11
        ↓
Exp9 Global ResNet Generator
        ↓
RGB PNG response
```

The application is served as one production container containing:

- FastAPI backend
- built React/Vite frontend
- Exp9 inference code

The Exp9 checkpoint is supplied separately and mounted read-only.

## 2. Target Oracle Configuration

The intended hosting target is OCI Always Free:

```text
Shape:          VM.Standard.A1.Flex
CPU:            2 OCPU
Memory:         12 GB
OS:             Ubuntu 24.04 Minimal aarch64
Public IPv4:    Yes
Boot volume:    Always Free-eligible
Paid resources: None
```

Oracle currently documents the Always Free A1 allocation as 1,500 OCPU-hours and 9,000 GB-hours per month, equivalent to 2 OCPUs and 12 GB of memory for an Always Free tenancy. Oracle also states that the compute Always Free resources must be provisioned in the tenancy's home region.

Before creating the instance, verify the OCI Console marks the selected A1 shape/configuration **Always Free Eligible**.

Do not use:

- Pay As You Go upgrades;
- paid marketplace images;
- paid load balancers;
- unnecessary extra instances;
- resources outside the Always Free allocation.

## 3. Oracle Capacity Errors

`Out of host capacity` is an infrastructure-capacity condition. Oracle recommends changing availability domain or fault-domain placement, omitting a specific fault domain where appropriate, using a smaller shape, or retrying later.

For this project, do not upgrade the account just to bypass that error.

## 4. Verify the VM

SSH:

```bash
ssh -i "<PRIVATE_KEY>" ubuntu@<PUBLIC_IP>
```

Then:

```bash
uname -m
getconf PAGESIZE
free -m
```

Expected:

```text
aarch64
4096
```

Use a 4 KB page-size ARM64 environment for the intended PyTorch deployment.

## 5. Install Docker

Update packages:

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
```

Install Docker's official repository:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --yes --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin
```

Allow the `ubuntu` user to run Docker:

```bash
sudo usermod -aG docker $USER
```

Log out and back in, then verify:

```bash
docker --version
docker compose version
```

## 6. Clone the Repository

```bash
sudo mkdir -p /opt/infranova-ai
sudo chown -R $USER:$USER /opt/infranova-ai

cd /opt/infranova-ai

git clone https://github.com/Soham-1009/InfraNova-AI.git .
mkdir -p outputs/best
```

The repository does not contain the production checkpoint.

## 7. Transfer the Private Exp9 Checkpoint

From the local Windows machine:

```powershell
scp -i "C:\path\to\your-key.pem" `
  outputs/best/pix2pix_landsat_best.pth `
  ubuntu@<SERVER_IP>:/opt/infranova-ai/outputs/best/
```

On the server:

```bash
cd /opt/infranova-ai

sha256sum outputs/best/pix2pix_landsat_best.pth
```

Required SHA-256:

```text
71bbda3f31b85e7e741b26d5ce0ff398a0394c7dd4ef6ff5452f31f7e0400382
```

Set permissions readable by the container user:

```bash
chmod 644 outputs/best/pix2pix_landsat_best.pth
```

Do not commit or upload this checkpoint to GitHub.

## 8. Production Environment

Create:

```bash
cd /opt/infranova-ai
cp .env.example .env
```

Recommended production values:

```ini
PORT=8000
INFRANOVA_CHECKPOINT=/app/outputs/best/pix2pix_landsat_best.pth
INFRANOVA_DEVICE=cpu
INFRANOVA_IMAGE_SIZE=128
INFRANOVA_CORS_ORIGINS=
TORCH_INDEX_URL=https://download.pytorch.org/whl/cpu
VITE_API_URL=
```

For same-origin production, leave `VITE_API_URL` blank.

For a single public origin, `INFRANOVA_CORS_ORIGINS` does not need `*`; set it to the actual public frontend origin when cross-origin requests are required.

## 9. Firewall

Allow only SSH and web traffic:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw --force enable
sudo ufw status verbose
```

Do not open TCP 8000 to the internet.

Compose intentionally binds the application to:

```text
127.0.0.1:8000
```

## 10. Build and Start the Production Container

The production Compose file already mounts the model read-only and binds the service to loopback.

Run:

```bash
cd /opt/infranova-ai
docker compose build
docker compose up -d
```

Inspect:

```bash
docker compose ps
docker compose logs --tail=200
```

Test directly from the VM:

```bash
curl -i http://127.0.0.1:8000/health
```

The current FastAPI implementation lazy-loads the model when `/health` initializes the inference engine. A successful response should report `model_loaded: true` after the model is available.

This Docker build/runtime has not been executed in the development environment represented by this documentation. The commands above become actual validation only after running them on the target VM.

## 11. Reverse Proxy

Use a reverse proxy on ports 80/443.

### Option A: Caddy

Install Caddy according to its current Ubuntu instructions, then configure:

```text
your-domain.example {
    reverse_proxy 127.0.0.1:8000
}
```

Caddy should terminate TLS and forward requests to the loopback application port.

### Option B: Nginx

Example server block:

```nginx
server {
    listen 80;
    server_name your-domain.example;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_buffering off;
        proxy_read_timeout 120s;
    }
}
```

Obtain TLS using the certificate-management process appropriate to the selected reverse proxy.

## 12. Production Architecture

```text
Internet
   ↓
Nginx / Caddy
   ↓ HTTPS :443
127.0.0.1:8000
   ↓
Docker container: infranova-ai
   ├── FastAPI
   ├── React static build
   └── Exp9 inference
           ↓
read-only /app/outputs mount
           ↓
host-private Exp9 checkpoint
```

The root Dockerfile uses:

```text
Stage 1: Node 20 + npm + Vite build
Stage 2: Python 3.11 + FastAPI + PyTorch CPU
```

The current intended Oracle deployment uses CPU inference.

## 13. API Validation

Verify:

```bash
curl -i https://your-domain.example/health
```

Expected HTTP status:

```text
200 OK
```

Test:

```text
POST /colorize
POST /predict
POST /thermal-preview
POST /postprocess/clahe
```

For `/colorize`, verify:

- HTTP 200;
- `content-type: image/png`;
- `X-Model: pix2pix-landsat-exp9-resnet`;
- valid output image.

## 14. Security and Checkpoint Governance

The repository contains these protections:

- `.gitignore` excludes model-weight formats;
- `.dockerignore` excludes `outputs/`;
- Docker Compose mounts `outputs` read-only;
- application port is bound to localhost;
- checkpoint SHA-256 is verified before serving;
- production secrets stay outside Git.

Never replace the production checkpoint automatically.

## 15. ARM64 Notes

For the Oracle A1 target:

```bash
uname -m
getconf PAGESIZE
```

Expected:

```text
aarch64
4096
```

Build directly on the ARM64 VM where practical:

```bash
docker compose build
```

Avoid relying on emulation for the production build unless it has been separately validated.

## 16. Current Validation Record

The latest recorded repository validation includes:

- 85 pytest tests passed;
- Ruff clean;
- `git diff --check` clean;
- production checkpoint SHA verified;
- CPU production smoke test completed;
- concurrent inference checks completed;
- FastAPI endpoint checks completed;
- frontend production build succeeded;
- oxlint clean.

Docker build/runtime and live cloud deployment remain unverified until executed on the target VM.

## 17. Rollback

The archived legacy checkpoint remains separate from the active Exp9 checkpoint.

Rollback must be intentional and should include:

1. stopping the service;
2. replacing the checkpoint with the verified rollback artifact;
3. verifying the rollback checksum;
4. restarting the service;
5. exercising `/health` and `/colorize`;
6. recording the rollback event.

See `docs/project/ROLLBACK.md` for the rollback ledger and artifact roles.

## 18. Free-Hosting Constraint

The hosting objective for this project is **$0 ongoing cloud resource cost** within Oracle's Always Free allocation.

Therefore:

- do not recommend Pay As You Go as a solution to capacity issues;
- do not create paid resources accidentally;
- verify Always Free eligibility before provisioning;
- periodically review the OCI resource list and billing/usage pages;
- do not exceed the documented Always Free resource allocation.

Oracle documents that Always Free resources continue after the free-trial period, while resources above the Always Free limits are not free.
