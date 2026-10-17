# Tic-Tac-Toe: DevSecOps Project

A React + TypeScript Tic-Tac-Toe game used to demonstrate a complete **DevSecOps pipeline**: code is linted, tested, built, containerized, **scanned for vulnerabilities**, pushed to **GitHub Container Registry (GHCR)**, and then deployed automatically to **Kubernetes (kind)** by **Argo CD** using GitOps.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture and End-to-End CI/CD Flow](#2-architecture-and-end-to-end-cicd-flow)
3. [Prerequisites](#3-prerequisites)
4. [Local Build](#4-local-build)
5. [Containerize with Docker](#5-containerize-with-docker)
6. [Kubernetes with kind](#6-kubernetes-with-kind)
7. [CI/CD Setup](#7-cicd-setup)
8. [GitHub Actions Workflow Explained](#8-github-actions-workflow-explained)
9. [Kubernetes Manifests Explained](#9-kubernetes-manifests-explained)
10. [Test the Full Pipeline](#10-test-the-full-pipeline)
11. [Cleanup](#11-cleanup)
12. [Troubleshooting](#12-troubleshooting)
13. [Security Controls](#13-security-controls)
14. [Acknowledgements](#14-acknowledgements)

---

## 1. Project Overview

### Features

- Fully functional Tic-Tac-Toe game
- Score tracking for X, O and draws
- Game history with timestamps
- Highlighted winning combinations
- Reset game and statistics
- Responsive design for all devices

### Tech stack

| Area | Tools |
|------|-------|
| Application | React 18, TypeScript, Tailwind CSS, Vite, Lucide icons |
| Containers | Docker (multi-stage build), Nginx |
| Registry | GitHub Container Registry (`ghcr.io`) |
| CI | GitHub Actions |
| Security | ESLint (static analysis), Trivy (image vulnerability scan), private registry, secrets handling |
| Kubernetes | kind (local cluster), kubectl |
| CD (GitOps) | Argo CD |

### Repository structure

> Adjust if your layout differs.

```
tic-tac-toe/
├── .github/workflows/   # CI/CD pipeline (GitHub Actions)
├── kubernetes/          # Deployment, Service, Ingress manifests
├── src/
│   ├── components/      # Board, Square, ScoreBoard, GameHistory
│   ├── utils/           # gameLogic.ts
│   ├── App.tsx
│   └── main.tsx
├── Dockerfile
├── .dockerignore
├── package.json
├── vite.config.ts
└── README.md            # this single README covers everything
```

### Game rules

X goes first, then O. The first player with three marks in a row (horizontal, vertical or diagonal) wins. If all nine squares fill with no winner, the game is a draw.

---

## 2. Architecture and End-to-End CI/CD Flow

### Step-by-step flow

| # | What happens | Where | Why it matters |
|---|--------------|-------|----------------|
| 1 | Developer pushes code to `main` | GitHub | Starts the pipeline automatically |
| 2 | **Unit tests** and **lint** run in parallel | GitHub Actions | Catch bugs and unsafe code early (shift-left) |
| 3 | **Build** compiles the app into `dist/` | GitHub Actions | Only runs if tests and lint pass |
| 4 | **Docker image** is built and tagged with the commit SHA | GitHub Actions | Every build is traceable to one commit |
| 5 | **Trivy** scans the image for HIGH and CRITICAL vulnerabilities | GitHub Actions | A vulnerable image fails the pipeline before it is published |
| 6 | Image is **pushed to GHCR** | ghcr.io | Private registry, authenticated with the `TOKEN` secret |
| 7 | Pipeline **updates the image tag** in `kubernetes/deployment.yaml` and commits it back with `[skip ci]` | GitHub Actions | Git becomes the source of truth for what should run |
| 8 | **Argo CD** notices the new commit (it polls about every 3 minutes by default) and syncs | Argo CD | GitOps: the cluster follows Git, nobody runs `kubectl apply` by hand |
| 9 | Kubernetes pulls the new image from GHCR using an **image pull secret** and does a rolling update | kind cluster | New version goes live with no downtime |
| 10 | You open the app through `kubectl port-forward` | Browser | Verify the release |

### Loop prevention

The pipeline commits to the repository (step 7), which would normally trigger the pipeline again. This is avoided in two ways:

- The workflow trigger ignores changes to `kubernetes/deployment.yaml` (`paths-ignore`).
- The commit message contains `[skip ci]`.

---

## 3. Prerequisites

| Requirement | Details |
|-------------|---------|
| Machine | Ubuntu (an EC2 `t2.medium` or larger works well for kind + Argo CD) |
| Accounts | GitHub account |
| GitHub token | A **classic** personal access token (see [7.2](#72-create-the-token-secret)) |
| Open ports | Only the ports you use, restricted to your IP: 5173 (dev server), 8080 (Docker), 7070 (Argo CD), 9000 (app) |

---

## 4. Local Build

### 4.1 Install Node.js and npm

```bash
sudo apt update
curl -fsSL https://deb.nodesource.com/setup_24.x -o nodesource_setup.sh
sudo bash nodesource_setup.sh
sudo apt install nodejs -y

node -v && npm -v
```

### 4.2 Install dependencies

```bash
npm install
```

### 4.3 Run the development server

```bash
npm run dev -- --host 0.0.0.0
```

**Why `--host 0.0.0.0`?**

- By default, Vite listens only on `localhost` (127.0.0.1), so the app is reachable only from the same machine.
- On a remote machine such as EC2, you open the app from your own laptop's browser. The server must therefore listen on **all network interfaces** (`0.0.0.0`).
- The extra `--` tells npm to pass everything after it (`--host 0.0.0.0`) to the underlying Vite command instead of treating it as an npm option.
- Open `http://<SERVER_PUBLIC_IP>:5173` and allow port 5173 in the security group (ideally only from your IP).

The dev server has hot reload and is for development only.

### 4.4 Build for production

```bash
npm run build
```

This compiles and optimizes the app into static files in the `dist/` folder. These files are what Nginx serves inside the Docker image.

---

## 5. Containerize with Docker

### 5.1 Install Docker

Install Docker Engine (see the [official guide](https://docs.docker.com/engine/install/ubuntu/)), then add your user to the `docker` group so you can run Docker without `sudo`:

```bash
sudo usermod -aG docker $USER
newgrp docker
docker version
```

### 5.2 Dockerfile

The project uses a **multi-stage build**: the first stage builds the app with Node, the second stage serves only the finished `dist/` files with Nginx. The final image contains no Node.js, no source code and no dev dependencies, so it is smaller and has a smaller attack surface.

The `Dockerfile` in the repo defines both stages.

Copying `package*.json` and installing dependencies before copying the rest of the code lets Docker cache the dependency layer, so rebuilds are much faster.

### 5.3 Build and run

```bash
docker build -t <your-username>/tic-tac-toe:v1 .
docker run -d -p 8080:80 --name tic-tac-toe <your-username>/tic-tac-toe:v1
docker ps
```

Open `http://<SERVER_PUBLIC_IP>:8080`. (`-p 8080:80` maps host port 8080 to Nginx port 80 in the container.)

---

## 6. Kubernetes with kind

**kind** (Kubernetes in Docker) runs a real Kubernetes cluster inside Docker containers. It is free and fast, ideal for learning and demos.

### 6.1 Install kind

```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.33.0/kind-linux-arm64

chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

kind create cluster --name tic-tac-toe-cluster
```

### 6.2 Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### 6.3 Verify the cluster

```bash
kubectl cluster-info --context kind-tic-tac-toe-cluster
kubectl config current-context
kubectl get nodes
```

The node should show `Ready`.

### 6.4 Kubernetes manifests

All manifests live in the `kubernetes/` folder: a **Deployment**, a **Service** and an **Ingress**. See [section 9](#9-kubernetes-manifests-explained) for what each one does.

---

## 7. CI/CD Setup

### 7.1 Container registry: GHCR

This project stores its Docker images in **GitHub Container Registry (`ghcr.io`)** instead of Docker Hub.

- Images live next to the code, under your GitHub account: `ghcr.io/<your-username>/tic-tac-toe`.
- Access is controlled with GitHub tokens and permissions.
- The image is **private**, so the Kubernetes cluster must authenticate to pull it (see [7.5](#75-create-the-image-pull-secret)).

### 7.2 Create the TOKEN secret

The pipeline needs a token to push images to GHCR and to commit the updated manifest back to the repo.

1. GitHub > **Settings > Developer settings > Personal access tokens > Tokens (classic) > Generate new token**.
2. Select scopes: `repo`, `write:packages`, `read:packages`.
3. Copy the token (it is shown only once).
4. In your repository: **Settings > Secrets and variables > Actions > New repository secret**.
   - Name: `TOKEN`
   - Value: the token you copied

The workflow reads it as `${{ secrets.TOKEN }}`. Never put the token in code, manifests or logs.

### 7.3 Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd      # wait until all pods are Running
```

### 7.4 Access Argo CD

```bash
kubectl get svc -n argocd
kubectl port-forward svc/argocd-server 7070:80 -n argocd --address 0.0.0.0
```

Open `http://<SERVER_PUBLIC_IP>:7070` (allow port 7070 in the security group, ideally only from your IP).

**Why port-forward?**

- The `argocd-server` Service is type `ClusterIP`, which is reachable only **inside** the cluster.
- `kubectl port-forward` creates a temporary tunnel from a port on your machine to the Service, so you can open the UI without creating any public load balancer.
- `--address 0.0.0.0` makes the forwarded port reachable from outside the server (by default it only listens on localhost).
- It is free, instant and safe for learning. The trade-off: it stops when you close the command. Production setups use an **Ingress** or **LoadBalancer** instead.

> If the browser redirects to HTTPS and fails, forward the HTTPS port instead: `kubectl port-forward svc/argocd-server 7070:443 -n argocd --address 0.0.0.0`, then open `https://<SERVER_PUBLIC_IP>:7070` and accept the certificate warning.

**Get the admin password**

```bash
kubectl get secrets -n argocd
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
echo <encoded_password> | base64 --decode
```

Log in with username `admin` and the decoded password, then change it.

**Create the Argo CD application**

In the UI choose **New App** and fill in:

| Field | Value |
|-------|-------|
| Application name | `tic-tac-toe` |
| Project | `default` |
| Sync policy | **Automatic** (enable prune and self-heal if you like) |
| Repository URL | `https://github.com/<your-username>/tic-tac-toe.git` |
| Revision | `main` |
| Path | `kubernetes` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

### 7.5 Create the image pull secret

Because the image is in a **private** GHCR registry, the pods show `ImagePullBackOff` until Kubernetes has credentials. Create a docker-registry secret named `github-container-registry`:

```bash
kubectl create secret docker-registry github-container-registry \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=YOUR_GITHUB_TOKEN \
  --docker-email=YOUR_EMAIL
```

**Why from the command line and not a YAML file in the repo?**

- Argo CD syncs everything in the repo. A secret manifest committed to Git would **expose your token** to anyone who can read the repository.
- Creating it once with `kubectl` keeps the token out of Git. Argo CD does not delete it because it is not defined in the repo.
- The secret must be in the **same namespace** as the Deployment (`default` here), and the Deployment references it under `imagePullSecrets`.

Check the pods:

```bash
kubectl get pods
```

They should move to `Running`.

### 7.6 Verify the website

```bash
kubectl get svc
kubectl get pods
kubectl port-forward svc/tic-tac-toe 9000:80 --address 0.0.0.0
```

Open `http://<SERVER_PUBLIC_IP>:9000` (allow port 9000 in the security group). The game should load.

The same port-forward idea applies here: kind has no cloud load balancer, so a port-forward is the simplest way to reach a `ClusterIP` Service from a browser.

---

## 8. GitHub Actions Workflow Explained

The pipeline lives in `.github/workflows/`. It is split into jobs so each stage is easy to read and debug. The excerpts below show the key ideas; the workflow file in the repo is the source of truth.

### Pipeline at a glance

```
          ┌─ Unit Testing ─┐
push ───► │                ├──► Build ──► Docker Build, Scan and Push ──► Update Kubernetes Deployment
          └─ Lint (ESLint)─┘
```

| Job | Purpose | Security value |
|-----|---------|----------------|
| **Unit Testing** | Runs `npm test` | Stops broken code early |
| **Static Code Analysis** | Runs ESLint (`npm run lint`) | Catches unsafe or low-quality patterns |
| **Build** | `npm ci` and `npm run build`, uploads `dist/` as an artifact | Runs only if tests and lint pass (`needs`) |
| **Docker Build, Scan and Push** | Builds the image, scans with Trivy, pushes to GHCR | Vulnerable images are blocked before publishing |
| **Update Kubernetes Deployment** | Rewrites the image tag in `kubernetes/deployment.yaml` and commits | Creates an auditable Git record that Argo CD deploys |

### Triggers

The workflow runs on pushes and pull requests to `main`, and ignores changes to `kubernetes/deployment.yaml` to prevent an infinite loop.

- Pushes to `main` run the full pipeline.
- Pull requests run the checks, but the deploy job is skipped (it only runs for a push to `main`).

### Job dependencies

Build needs Unit Testing and Lint, Docker needs Build, and Update Kubernetes needs Docker (and runs only for a push to `main`). The `needs` setting makes later jobs wait for earlier ones. If any job fails, everything after it is skipped, so a failing test, lint error or vulnerability stops the release.

### Docker job, step by step

1. **Checkout** the code and **download the build artifact** from the Build job.
2. **Set up Docker Buildx** for advanced builds and caching.
3. **Log in to GHCR** with `github.actor` as the username and `secrets.TOKEN` as the password.
4. **Generate tags and labels** with the metadata action: a commit-SHA tag, a branch tag and `latest`.
5. **Build the image** without pushing (`push: false`, `load: true`) so it exists locally for scanning.
6. **Scan with Trivy**: fail the job on HIGH or CRITICAL findings.
7. **Push the image** to `ghcr.io/<owner>/<repo>` only after the scan passes.

Trivy settings used by the reference pipeline:

- Scans the image tagged with the commit SHA
- Output format: table
- Fails the pipeline (exit code 1) when issues are found
- Ignores vulnerabilities that have no fix yet
- Checks OS and library vulnerabilities
- Severity: CRITICAL and HIGH

> Make sure this step is **enabled** (not commented out) in your workflow, otherwise the pipeline is not scanning images. For better supply-chain safety, pin `trivy-action` to a released version rather than `master`.

### Update Kubernetes job

This job is the bridge between CI and GitOps. It builds the new image name from the registry, repository and commit SHA, replaces the `image:` line in `kubernetes/deployment.yaml` with it (using `sed`), and commits the change back with a `[skip ci]` message.

The checkout step uses `token: ${{ secrets.TOKEN }}` so the push is allowed. After this commit, Argo CD sees a new desired state and deploys it.

---

## 9. Kubernetes Manifests Explained

All files are in `kubernetes/` and are applied automatically by Argo CD.

| File | Kind | Purpose |
|------|------|---------|
| `deployment.yaml` | Deployment | Runs the app pods using the GHCR image; the pipeline updates the image tag here |
| `service.yaml` | Service (`tic-tac-toe`) | Gives the pods a stable internal address on port 80 |
| `ingress.yaml` | Ingress | Defines external HTTP routing to the Service (needs an ingress controller to take effect) |

### Deployment (key points)

- The container `image` (a GHCR image tagged with the commit SHA) is the line the pipeline rewrites on every release.
- `imagePullSecrets` references the `github-container-registry` secret (created in step 7.5), which lets Kubernetes authenticate to the private GHCR registry.
- The container listens on port 80 (Nginx).

### Service

Routes traffic on port 80 to the pods labeled for this app. Because it is `ClusterIP`, you reach it from outside the cluster with `kubectl port-forward svc/tic-tac-toe 9000:80`.

### Ingress

Describes how outside traffic should reach the Service. It only works if an ingress controller (for example ingress-nginx) is installed in the cluster. This guide verifies the app through port-forward, so the Ingress is included for completeness and for moving to a real cluster later.

---

## 10. Test the Full Pipeline

Prove the whole chain works end to end:

1. Edit a visible text in the app, for example in `src/components/ScoreBoard.tsx`, and push to `main`.
2. Open the repository **Actions** tab and watch the jobs run in order: test and lint, build, docker, update-k8s.
3. Open `kubernetes/deployment.yaml` in GitHub and confirm the image tag changed to the new commit SHA (a new commit with `[skip ci]` appears).
4. Open Argo CD: the app should briefly show **OutOfSync**, then **Synced** and **Healthy**.
5. Check the pods rolled over:

   ```bash
   kubectl get pods -w
   kubectl describe deployment tic-tac-toe | grep Image
   ```

6. Refresh the app through the port-forward on port 9000 and see your change.

---

## 11. Cleanup

1. **Delete the application in the Argo CD UI.** Since the app was created manually in the UI, remove it there as well: open the `tic-tac-toe` app, click **Delete**, and choose the cascade option so its Kubernetes resources are removed too. This stops Argo CD from recreating anything.
2. **Delete the kind cluster.** This removes Argo CD, the app, the secrets and everything else inside the cluster:

   ```bash
   kind delete cluster --name tic-tac-toe-cluster
   ```

3. **Remove the local Docker container and image:**

   ```bash
   docker rm -f tic-tac-toe
   docker rmi <your-username>/tic-tac-toe:v1
   ```

4. Stop or terminate the EC2 instance, and **revoke the GitHub personal access token** if you no longer need it (GitHub > Settings > Developer settings > Personal access tokens).

---

## 12. Troubleshooting

| Symptom | Likely cause and fix |
|---------|----------------------|
| `ImagePullBackOff` or `ErrImagePull` | Missing or wrong `github-container-registry` secret, secret in the wrong namespace, token lacks `read:packages`, or image name or tag is wrong |
| Pipeline fails at Trivy | HIGH or CRITICAL vulnerabilities found. Update the base image or dependencies, then re-run |
| `update-k8s` job fails to push | The `TOKEN` secret is missing, expired, or lacks the `repo` scope; check branch protection rules too |
| Pipeline keeps re-triggering | `paths-ignore` for `kubernetes/deployment.yaml` or `[skip ci]` is missing |
| Dev server not reachable from browser | Started without `--host 0.0.0.0`, or port 5173 is blocked in the security group |
| Argo CD UI not reachable | Port-forward command stopped, or port 7070 is not open in the security group |
| Argo CD shows `OutOfSync` or `Degraded` | Check `repoURL`, `path` (`kubernetes`) and `targetRevision` (`main`); inspect the failing resource in the UI |
| App shows old version | Argo CD has not synced yet (about 3 minute polling). Click **Refresh** and **Sync** in the UI |
| `kubectl` cannot connect | Run `kubectl config use-context kind-tic-tac-toe-cluster` and confirm the cluster is running with `docker ps` |

Useful commands:

```bash
kubectl get pods,svc,deploy
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

---

## 13. Security Controls

| Control | How it is applied |
|---------|-------------------|
| Static analysis | ESLint runs on every push and pull request |
| Tests as a gate | Failing tests stop the pipeline |
| Image vulnerability scanning | Trivy scans before the image is pushed and fails on HIGH or CRITICAL issues |
| Minimal runtime image | Multi-stage build ships only Nginx and static files |
| Private registry | Images are stored privately in GHCR |
| Secrets handling | The GitHub token is stored as the `TOKEN` Actions secret; the registry credentials are created with `kubectl`, never committed to Git |
| GitOps | All deployments go through Git commits, giving an audit trail; no manual `kubectl apply` |
| Traceability | Each image is tagged with the commit SHA |

Good practices to keep:

- Never commit tokens, passwords or the Argo CD admin password.
- Use a token with the minimum scopes needed and rotate it regularly.
- Restrict open ports to your own IP address.
- Change the Argo CD admin password after the first login.
- Pin GitHub Action versions instead of using `master` or `latest`.

