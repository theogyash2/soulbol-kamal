# Kamal Deployment System

Deploy a Go web app (Web + Worker) to Ubuntu EC2 using Kamal + GitHub Actions CI/CD.

`Go` | `Ubuntu EC2` | `AWS ECR` | `GitHub Actions`

Repository: https://github.com/theogyash2/kamal-deploy-demo

---

## The Files

Only the config files need to be changed per project.

| File | Type | Description |
|------|------|-------------|
| `config/deploy.yml` | Change accordingly | Main Kamal config — hosts, services, registry, proxy |
| `.kamal/secrets` | Change accordingly | Environment variable references for Kamal |
| `.github/workflows/deploy.yml` | Don't change | CI/CD |

---

## How It Works

A single Docker image is built from the Dockerfile. At runtime, the `APP_ROLE` environment variable decides what the container does:

- `APP_ROLE=web` — starts the HTTP web server on port 3000
- `APP_ROLE=worker` — starts the background worker process

Both services run on the same EC2 instance. Kamal manages them as two separate containers (Web and Worker) from the same image. The Kamal proxy sits in front and routes external traffic only to the Web container on port 3000.

---

## Architecture

```
Internet
   |
   v
Port 80 / 443  -->  kamal-proxy
                        |
                        v
                   Web container  (APP_ROLE=web)   :3000

Worker container  (APP_ROLE=worker)  :background  [no external traffic]
```

The Worker container runs an internal HTTP server on port 3000 (inside its own container) only for status checks. The Web container queries it directly using the server IP and internal port.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| EC2 Instance | Ubuntu, public IP, SSH key (.pem file) |
| AWS ECR repo | Image registry |
| IAM User/credentials | IAM user with AWS CLI access and specific permissions |
| SSH Key file | SSH key to put in GitHub Secrets |

---

## Step 1 — EC2 Instance Setup

Create an EC2 Instance with the following configurations:

### 1.1 Security Group — Open Ports

EC2 -> Security Groups -> Inbound Rules -> Add:

| Port | Purpose |
|------|---------|
| 22 | SSH — Kamal connects here to deploy |
| 80 | HTTP traffic via Kamal proxy |
| 443 | HTTPS traffic (if using SSL + domain) |

### 1.2 Optional (User Data)

When creating the EC2 instance, paste this into the User Data field under Advanced Details. It runs once on first boot and enables root SSH using the same key as the ubuntu user — required for Kamal.

```yaml
#cloud-config
runcmd:
  - sudo mkdir -p /root/.ssh
  - sudo chmod 700 /root/.ssh
  - sudo cp /home/ubuntu/.ssh/authorized_keys /root/.ssh/authorized_keys.tmp
  - sudo sed 's/^no-port-forwarding[^"]*exit 142" //' /root/.ssh/authorized_keys.tmp > /root/.ssh/authorized_keys
  - sudo chmod 600 /root/.ssh/authorized_keys
  - sudo chown root:root /root/.ssh/authorized_keys
  - sudo rm -f /root/.ssh/authorized_keys.tmp
```

> If you skip the User Data step, the CI/CD workflow includes a fallback step that does this automatically on first deploy.

### 1.3 SSH Key

Create a new key-pair or use the existing one, make sure you have downloaded it to your local file system.

---

## Step 2 — Dockerfile (According to the project)

The Dockerfile uses a two-stage build. Stage 1 compiles the Go binary; Stage 2 copies only the binary into a minimal Alpine image. This keeps the final image small.

```dockerfile
# Stage 1: Build
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o my-go-app .

# Stage 2: Production image
FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/my-go-app .
EXPOSE 3000
ENTRYPOINT ["/app/my-go-app"]
```

The ENTRYPOINT is fixed to the binary. The CMD (and therefore the APP_ROLE) is set per-service in `config/deploy.yml`, not in the Dockerfile.

---

## Step 3 — App Role Pattern (main.go) (According to the project)

The app reads the `APP_ROLE` environment variable on startup to decide which mode to run in:

```go
appRole := os.Getenv("APP_ROLE")
if appRole == "worker" {
    startWorker()
} else {
    startWebServer()
}
```

This pattern means one image, one Dockerfile, one build — but two independently deployable and scalable services controlled purely by environment variables.

---

## Step 4 — config/deploy.yml

CD into the project and run `gem install kamal` and then `kamal init` — it will generate `config/deploy.yml` and a secrets file.

Replace the IP, ECR URL, and SSH key path with your own values.

**Without domain (IP only):**

```yaml
service: soulbol-server
image: soulbol-server-kamal

builder:
  arch: amd64

servers:
  web:
    hosts:
      - 13.207.60.18
    env:
      APP_ROLE: web
    proxy:
      path_prefix: /

  Worker:
    hosts:
      - 13.207.60.18
    env:
      APP_ROLE: worker
    cmd: /app/my-go-app
    proxy: false

ssh:
  user: root
  keys:
    - ~/.ssh/kamal-test-new.pem

proxy:
  # ssl: true
  host: 13.207.60.18
  app_port: 3000
  healthcheck:
    path: /up
    interval: 1
    timeout: 3

registry:
  server: 495599768662.dkr.ecr.ap-south-1.amazonaws.com
  username: AWS
  password:
    - KAMAL_REGISTRY_PASSWORD
```

**With domain (SSL):**

```yaml
service: soulbol-server
image: soulbol-server-kamal

builder:
  arch: amd64

servers:
  web:
    hosts:
      - 13.207.60.18
    env:
      APP_ROLE: web
    proxy:
      path_prefix: /

  Worker:
    hosts:
      - 13.207.60.18
    env:
      APP_ROLE: worker
    cmd: /app/my-go-app
    proxy: false

ssh:
  user: root
  keys:
    - ~/.ssh/kamal-test-new.pem

proxy:
  ssl: true
  host: api.soulbol.com
  app_port: 3000
  healthcheck:
    path: /up
    interval: 1
    timeout: 3

registry:
  server: 495599768662.dkr.ecr.ap-south-1.amazonaws.com
  username: AWS
  password:
    - KAMAL_REGISTRY_PASSWORD
```

**Keywords reference:**

| Keyword | Description |
|---------|-------------|
| `service: soulbol-server` | Unique name — used to name containers and services |
| `image: kamal` | ECR repo name |
| `arch: amd64` | Build for Linux amd64 (standard EC2) |
| `hosts: - 13.126.175.228` | Your EC2 public IP |
| `APP_ROLE: web` | Tells the container to run as web server |
| `path_prefix: /` | Route all traffic to this service |
| `APP_ROLE: worker` | Tells the container to run as background worker |
| `cmd: /app/my-go-app` | Explicit CMD override (optional) |
| `proxy: false` | Worker gets no external traffic |
| `user: root` | Always root — never add inline comments here |
| `host: 13.126.175.228` | Or your domain name if using SSL |
| `path: /up` | Kamal polls this before routing traffic |
| `- KAMAL_REGISTRY_PASSWORD` | Read from .kamal/secrets |

---

## Step 5 — .kamal/secrets

This file tells Kamal which environment variables to read. The value gets substituted from the shell environment at deploy time.

```
KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD
```

> Do not put the actual token here. The CI/CD pipeline generates it fresh on every deploy using the AWS CLI.

---

## Step 6 — CI/CD: .github/workflows/deploy.yml

It handles ECR authentication, SSH setup, root access provisioning, and the Kamal deploy automatically.

```yaml
name: Deploy with Kamal

on:
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Ruby & Kamal
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: 3.3
          bundler-cache: true

      - name: Install Kamal
        run: gem install kamal

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-south-1

      - name: Get ECR password
        run: |
          ECR_PASSWORD=$(aws ecr get-login-password --region ap-south-1)
          echo "::add-mask::$ECR_PASSWORD"
          echo "KAMAL_REGISTRY_PASSWORD=$ECR_PASSWORD" >> $GITHUB_ENV

      - name: Set up SSH key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.SSH_PRIVATE_KEY }}" > ~/.ssh/kamal-test-new.pem
          chmod 600 ~/.ssh/kamal-test-new.pem

      # - name: Reboot the proxy with Kamal
      #   run: kamal proxy reboot -y
      #   env:
      #     KAMAL_REGISTRY_PASSWORD: ${{ env.KAMAL_REGISTRY_PASSWORD }}

      - name: Setup root SSH access on instance
        run: |
          SERVER_IP=$(grep -A2 'hosts:' config/deploy.yml | grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | head -1)
          ssh -i ~/.ssh/kamal-test-new.pem -o StrictHostKeyChecking=no ubuntu@$SERVER_IP bash << 'EOF'
            sudo mkdir -p /root/.ssh
            sudo chmod 700 /root/.ssh
            sudo cp /home/ubuntu/.ssh/authorized_keys /root/.ssh/authorized_keys.tmp
            sudo sed 's/^no-port-forwarding[^"]*exit 142" //' /root/.ssh/authorized_keys.tmp | sudo tee /root/.ssh/authorized_keys
            sudo chmod 600 /root/.ssh/authorized_keys
            sudo chown root:root /root/.ssh/authorized_keys
            sudo rm -f /root/.ssh/authorized_keys.tmp
          EOF

      - name: Deploy with Kamal
        run: |
          kamal setup
          kamal deploy
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ env.KAMAL_REGISTRY_PASSWORD }}
```

---

## Step 7 — GitHub Secrets

Go to your repo: Settings -> Secrets and variables -> Actions -> New repository secret.

| Secret Name | Value |
|-------------|-------|
| `AWS_ACCESS_KEY_ID` | IAM user access key ID |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret access key |
| `SSH_PRIVATE_KEY` | Full contents of your .pem file including `-----BEGIN` and `-----END` lines |

---

## Step 8 — IAM Permissions

The IAM user only needs the following permissions. Policy and user already created: `kamal-deploy-soulbol` and `kamal-soulbol-test`. Just attach the policy to the user.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ecr:GetAuthorizationToken",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetDownloadUrlForLayer",
                "ecr:BatchGetImage",
                "ecr:InitiateLayerUpload",
                "ecr:UploadLayerPart",
                "ecr:CompleteLayerUpload",
                "ecr:PutImage",
                "ecr:CreateRepository",
                "ecr:DescribeRepositories"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances",
                "ec2:DescribeInstanceStatus"
            ],
            "Resource": "*"
        }
    ]
}
```

---

## What You Get

| Feature | Details |
|---------|---------|
| Auto-deploy | (Currently disabled) Every git push to main triggers a full build and deploy |
| Web + Worker from one image | APP_ROLE env var determines which service the container runs |
| Zero downtime | Old container stays live until the new one passes health checks |
| ECR image registry | Docker images stored and versioned in AWS ECR |
| Root SSH auto-provisioning | CI/CD sets up root access on new instances automatically |
| Secret masking | ECR token is masked in logs using `::add-mask::` |
| IP auto-detection | Workflow reads server IP from config/deploy.yml — no hardcoding |

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| Authentication failed for user root (always go for...) | Remove the inline comment from ssh.user — use just: `user: root` |
| servers/Web/env: should be a hash | Remove dashes from env values. Use `KEY: value` not `- KEY: value` |
| docker: command not found | Run `kamal setup` before `kamal deploy` — it installs Docker |
| kamal-proxy version too old | Run once: `kamal proxy reboot` |
| flag needs an argument: 'p' in -p | KAMAL_REGISTRY_PASSWORD is empty — check the Get ECR password step runs and the `::add-mask::` line is present |
| Secret 'KAMAL_REGISTRY_PASSWORD' not found in .kamal/secrets | Add `KAMAL_REGISTRY_PASSWORD=` (blank value) to `.kamal/secrets` and commit it |
| Permission denied (publickey) | SSH_PRIVATE_KEY secret must contain the full .pem file including header and footer |
| Please login as user ubuntu rather than root | Instance was recreated — root SSH not set up yet. The CI/CD setup step will fix it on next push |
| GitHub push blocked — secrets detected | Remove secrets from history with `git filter-repo`, or delete and reinit the repo |

---

## New Project Checklist

- [ ] Create EC2 instance — add User Data script for root SSH access
- [ ] Open ports 22, 80, 443 in Security Group
- [ ] Create ECR repository in AWS
- [ ] Write your app — read APP_ROLE env var to switch between web and worker
- [ ] Add Dockerfile with multi-stage build
- [ ] Run `kamal init` — generates `config/deploy.yml` and `.kamal/secrets`
- [ ] Fill in `config/deploy.yml` — IP, ECR URL, service names, app port
- [ ] Add `KAMAL_REGISTRY_PASSWORD=$KAMAL_REGISTRY_PASSWORD` to `.kamal/secrets`
- [ ] Add GitHub Secrets: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `SSH_PRIVATE_KEY`
- [ ] Add `.github/workflows/deploy.yml`
