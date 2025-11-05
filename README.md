# CI/CD Starter for Microservices (No Tests)

This starter gives you **build & deploy** pipelines with **GitHub Actions** for a microservices repo.
It supports two deployment targets—pick one or keep both:

1) **Kubernetes (Kustomize)** — production‑grade, with rolling updates.
2) **Single VM via SSH + docker-compose** — simple, for one host.

> This template intentionally **omits tests** per your request. You can add them later.

---

## Directory layout
```
.
├─ services/
│  ├─ service-a/
│  │  └─ Dockerfile
│  └─ service-b/
│     └─ Dockerfile
├─ k8s/
│  ├─ base/
│  │  ├─ namespace.yaml
│  │  ├─ deployment.yaml
│  │  └─ service.yaml
│  └─ overlays/
│     └─ prod/
│        ├─ kustomization.yaml
│        ├─ patch-deployment.yaml
│        └─ namespace.yaml
├─ docker-compose.prod.yml
├─ scripts/
│  ├─ list_changed_services.sh
│  └─ image_tag.sh
└─ .github/workflows/
   ├─ docker-build-push.yml
   ├─ deploy-k8s.yml
   └─ deploy-vm.yml
```

---

## Quick start

### 0) Prerequisites
- Each microservice lives under `services/<name>/` and has a `Dockerfile`.
- You push to GitHub.
- You have a container registry:
  - **GHCR** (recommended): `ghcr.io/<org-or-user>`; or
  - **Docker Hub**.

### 1) Configure GitHub Secrets
Go to **Repo Settings → Secrets and variables → Actions → New repository secret** and add:

For **image build** (both targets):
- `REGISTRY_USERNAME` — your registry username.
- `REGISTRY_PASSWORD` — a Personal Access Token or password.
- `REGISTRY` — e.g. `ghcr.io/your-user-or-org`
- `IMAGE_NAMESPACE` — e.g. your GitHub username or org (for ghcr.io it’s usually the same).

For **Kubernetes** (if using deploy‑k8s):
- `KUBE_CONFIG` — base64 of your kubeconfig file (run `base64 -w0 ~/.kube/config` on Linux/macOS).

For **VM deploy** (if using deploy‑vm):
- `SSH_HOST` — your VM host (e.g. `1.2.3.4`).
- `SSH_USER` — ssh user (e.g. `ubuntu`).
- `SSH_KEY` — base64 of your private key (e.g. `base64 -w0 ~/.ssh/id_rsa`).

### 2) Push this repo
Commit & push everything. Pipelines will:
- Build & push images for **only changed services** to the registry on PRs and pushes.
- Deploy on **push to `main`** via K8s or VM, depending on which workflow is enabled.

### 3) Customize
- Add more services under `services/` with a `Dockerfile`.
- Edit `k8s/base/deployment.yaml` names/ports and duplicate for each service (or split manifests per service).
- Update `docker-compose.prod.yml` to include your services and env.

---

## How it works

### Build & Push (docker-build-push.yml)
- Detects changed services by diffing `services/*` paths.
- Builds with Buildx and pushes to `${{ secrets.REGISTRY }}/${{ secrets.IMAGE_NAMESPACE }}/<service>:<sha>`.

### Deploy to Kubernetes (deploy-k8s.yml)
- Installs `kubectl` & `kustomize`.
- Replaces image tag in Kustomize overlay to the current commit SHA.
- Applies `kustomize build k8s/overlays/prod | kubectl apply -f -`.

### Deploy to a VM (deploy-vm.yml)
- SSH into the VM.
- Writes `.env` and `docker-compose.prod.yml`.
- Performs `docker compose pull && docker compose up -d`.

---

## Add a new service

1. Create `services/<your-service>/Dockerfile`.
2. Reference the image in:
   - Kubernetes: add a container block for it (or split into per‑service manifests).
   - VM: add a service to `docker-compose.prod.yml`.
3. Commit & push. The CI picks it up automatically if files under `services/<your-service>` change.

---

## Notes
- **No tests** included by design. You can add test steps before the build stage later.
- For monorepos you can also split Deploy workflows per service by using `paths` filters or multiple jobs.
