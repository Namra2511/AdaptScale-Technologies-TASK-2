# CI/CD Pipeline — GitHub Actions + AWS EC2

A Node.js web application with a fully automated CI/CD pipeline. Every push to `main` builds, tests and deploys the app to an AWS EC2 instance without any manual steps.

## Architecture

```
Developer → GitHub (main branch)
               ↓
         GitHub Actions
          ├── Install deps
          ├── Run tests
          └── SSH into EC2
                 ↓
           EC2 (Ubuntu 24.04)
            ├── git clone (fresh)
            ├── npm install
            └── pm2 reload
```

| Layer | Technology |
|---|---|
| App runtime | Node.js 18 + Express |
| CI/CD | GitHub Actions |
| Cloud | AWS EC2 (Ubuntu 24.04 LTS) |
| Process manager | PM2 |
| Reverse proxy | Nginx (port 80 → 3000) |

## Project Structure

```
.github/workflows/deploy.yml   # pipeline definition
public/index.html              # frontend — shows live deploy info
server.js                      # Express server
package.json
.gitignore
```

## How the Pipeline Works

1. Push to `main` triggers the workflow automatically
2. `build-and-test` job — installs deps and runs `npm test` on a GitHub-hosted runner
3. If tests pass, `deploy` job SSHs into the EC2 instance
4. Existing app folder is deleted and a fresh clone is pulled
5. Dependencies are installed and PM2 reloads the app
6. A health check hits `/health` — pipeline fails if the app isn't responding

The deployed webpage shows the **commit SHA** and **deploy timestamp**, which change on every push — making it easy to verify that auto-deployment is working.

## Setup

### 1. EC2 Instance

Launch an Ubuntu 24.04 EC2 instance (t2.micro) with the following inbound rules:

| Port | Purpose |
|---|---|
| 22 | SSH |
| 80 | HTTP (Nginx) |
| 3000 | Node.js direct |

### 2. EC2 Prerequisites

SSH into the instance and install the required tools once:

```bash
# Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs git

# PM2
sudo npm install -g pm2
pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

> The pipeline also installs Node.js and PM2 automatically if they are missing, so this step is optional.

### 3. GitHub Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Value |
|---|---|
| `EC2_SSH_KEY` | Contents of your `.pem` key file (including header/footer lines) |
| `EC2_HOST` | Public IPv4 address of the EC2 instance |
| `EC2_USER` | `ubuntu` |

### 4. Deploy

```bash
git push origin main
```

The pipeline runs automatically. The app will be live at `http://<EC2_HOST>:3000`.

## API Endpoints

| Endpoint | Description |
|---|---|
| `GET /` | Frontend dashboard |
| `GET /health` | Returns `{ status, hostname, uptime }` — used by pipeline |
| `GET /api/info` | Returns app version, environment, deploy time |
