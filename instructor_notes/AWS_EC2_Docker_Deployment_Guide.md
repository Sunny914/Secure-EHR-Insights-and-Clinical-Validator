# AWS EC2 Docker Deployment Guide

 **AWS EC2 Ubuntu instance using Docker**.

The deployment flow is:

**GitHub → EC2 → Docker Image → Docker Container → Streamlit UI**


---

## 1. Create an AWS EC2 Instance

Recommended configuration:

- OS: **Ubuntu 24.04 LTS**
- Instance type: **t3.large**
- CPU: 2 vCPU
- RAM: 8 GB
- Storage: 30–40 GB gp3
- Public IPv4: Enabled

### Security Group

Add these inbound rules:

| Type | Port | Source |
|---|---:|---|
| SSH | 22 | My IP |
| Custom TCP | 8501 | 0.0.0.0/0 |

You do **not** need to expose port `8000`.

FastAPI will communicate internally with Streamlit inside the Docker container.

---

## 2. Connect to EC2

From your local terminal:

```bash
ssh -i "FDE-2-yt.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Example:

```bash
ssh -i "FDE-2-yt.pem" ubuntu@54.123.45.67
```

All remaining commands should be executed inside EC2.

---

## 3. Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

Install required utilities:

```bash
sudo apt install -y ca-certificates curl git nano
```

---

## 4. Install Docker

Create the Docker keyring directory:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

Download Docker's GPG key:

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
```

Set permissions:

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Add Docker's official repository:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF2
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF2
```

Update packages:

```bash
sudo apt update
```

Install Docker:

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable Docker:

```bash
sudo systemctl enable --now docker
```

Add your user to the Docker group:

```bash
sudo usermod -aG docker $USER
```

Activate the Docker group:

```bash
newgrp docker
```

Verify Docker:

```bash
docker --version
```

Test Docker:

```bash
docker run --rm hello-world
```

---

## 5. Clone the GitHub Repository

Go to your home directory:

```bash
cd ~
```

Clone the repository:

```bash
git clone YOUR_GITHUB_REPOSITORY_URL FDE-Project-2
```

Example:

```bash
git clone https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git FDE-Project-2
```

Move into the project:

```bash
cd FDE-Project-2
```

Check files:

```bash
ls
```

You should see files similar to:

```text
requirements.txt
src
scripts
mentor-docs
.gitignore
```

---

## 6. Create the `.env` File

The `.env` file is ignored by Git, so it will not be downloaded when you clone the repository.

Create it manually:

```bash
nano .env
```

Add your values:

```env
DB_HOST=YOUR_DATABASE_HOST
DB_PORT=5432
DB_NAME=YOUR_DATABASE_NAME
DB_USER=YOUR_DATABASE_USER
DB_PASSWORD=YOUR_DATABASE_PASSWORD

OPENAI_API_KEY=YOUR_DEEPSEEK_API_KEY
OPENAI_BASE_URL=https://api.deepseek.com/v1
OPENAI_API_BASE=https://api.deepseek.com/v1
```

Save the file:

```text
Ctrl + O
Enter
Ctrl + X
```

Protect the file:

```bash
chmod 600 .env
```

Verify:

```bash
ls -la .env
```

Never commit the `.env` file to GitHub.

---

## 7. Create `.dockerignore`

Run:

```bash
cat > .dockerignore <<'EOF2'
.git
.gitignore
.env
*.env
__pycache__
*.pyc
*.pyo
.venv
venv
env
data
mentor-docs
EOF2
```

Verify:

```bash
cat .dockerignore
```

---

## 8. Create the Startup Script

The project needs two services:

- FastAPI → `8000`
- Streamlit → `8501`

Create `start.sh`:

```bash
cat > start.sh <<'EOF2'
#!/bin/sh
set -e

echo "Starting FastAPI..."

uvicorn src.api.main:app \
  --host 0.0.0.0 \
  --port 8000 &

API_PID=$!

echo "Waiting for FastAPI to become ready..."

while ! curl -fsS http://127.0.0.1:8000/openapi.json >/dev/null 2>&1; do
  if ! kill -0 "$API_PID" 2>/dev/null; then
    echo "FastAPI failed to start."
    wait "$API_PID"
    exit 1
  fi
  sleep 5
done

echo "FastAPI is ready."
echo "Starting Streamlit..."

exec streamlit run src/ui/app.py \
  --server.address=0.0.0.0 \
  --server.port=8501 \
  --server.headless=true
EOF2
```

Make it executable:

```bash
chmod +x start.sh
```

Verify:

```bash
cat start.sh
```

---

## 9. Create the Dockerfile

Create the file:

```bash
cat > Dockerfile <<'EOF2'
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PIP_NO_CACHE_DIR=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libpq-dev \
    curl \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .

RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN chmod +x /app/start.sh

EXPOSE 8000
EXPOSE 8501

CMD ["/app/start.sh"]
EOF2
```

Verify:

```bash
cat Dockerfile
```

---

## 10. Build the Docker Image

From the project directory:

```bash
docker build --pull -t fde-project-2:latest .
```

Check images:

```bash
docker images
```

You should see:

```text
fde-project-2
```

---

## 11. Run the Docker Container

Remove any previous container:

```bash
docker rm -f fde-project-2 2>/dev/null || true
```

Run the application:

```bash
docker run -d \
  --name fde-project-2 \
  --restart unless-stopped \
  --env-file .env \
  -p 8501:8501 \
  fde-project-2:latest
```

Only Streamlit port `8501` is exposed publicly.

FastAPI remains available internally on port `8000`.

---

## 12. Check Container Status

```bash
docker ps
```

View logs:

```bash
docker logs -f fde-project-2
```

Press:

```text
Ctrl + C
```

to exit the log viewer. This does not stop the container.

---

## 13. Test Streamlit

From EC2:

```bash
curl http://localhost:8501/_stcore/health
```

Expected:

```text
ok
```

---

## 14. Test FastAPI

Run:

```bash
docker exec fde-project-2 \
  curl -f http://127.0.0.1:8000/openapi.json >/dev/null \
  && echo "FastAPI is working"
```

Expected:

```text
FastAPI is working
```

---

## 15. Get EC2 Public IP

Run:

```bash
curl -s https://checkip.amazonaws.com
```

Example:

```text
54.123.45.67
```

---

## 16. Open the Application

Open in your browser:

```text
http://YOUR_EC2_PUBLIC_IP:8501
```

Example:

```text
http://54.123.45.67:8501
```

Your Streamlit application should now be available publicly.

---

# Deployment Architecture

```text
                    Internet
                       |
                       v
              AWS Security Group
                       |
                    :8501
                       |
                       v
             +--------------------+
             | Docker Container   |
             |                    |
Browser ---> | Streamlit :8501    |
             |       |            |
             |       v            |
             | FastAPI :8000      |
             |       |            |
             |       v            |
             | PostgreSQL /       |
             | pgvector Database  |
             |                    |
             | Presidio           |
             | NeMo Guardrails    |
             | ModernBERT         |
             +--------------------+
                       |
                       v
                    DeepSeek
```

---

# Database Networking Note

If PostgreSQL is hosted outside this EC2 instance, you do not need to expose port `5432` on your application EC2 server.

If you are using AWS RDS:

- Open PostgreSQL port `5432` on the **RDS Security Group**
- Allow traffic from the **EC2 Security Group**
- Do not expose PostgreSQL publicly unless absolutely required

---

# Useful Docker Commands

## Check containers

```bash
docker ps
```

## Check all containers

```bash
docker ps -a
```

## View logs

```bash
docker logs -f fde-project-2
```

## Restart container

```bash
docker restart fde-project-2
```

## Stop container

```bash
docker stop fde-project-2
```

## Start container

```bash
docker start fde-project-2
```

## Remove container

```bash
docker rm -f fde-project-2
```

## Check Docker images

```bash
docker images
```

---

# Redeploy After Updating GitHub

Whenever you update your project in GitHub, connect to EC2 and run:

```bash
cd ~/FDE-Project-2
```

Pull the latest code:

```bash
git pull
```

Rebuild the image:

```bash
docker build -t fde-project-2:latest .
```

Remove the old container:

```bash
docker rm -f fde-project-2
```

Start the new container:

```bash
docker run -d \
  --name fde-project-2 \
  --restart unless-stopped \
  --env-file .env \
  -p 8501:8501 \
  fde-project-2:latest
```

Check logs:

```bash
docker logs -f fde-project-2
```

---

# Complete Quick Deployment Commands

After Docker is installed, the essential flow is:

```bash
git clone YOUR_GITHUB_REPOSITORY_URL FDE-Project-2
cd FDE-Project-2
nano .env
docker build -t fde-project-2:latest .
docker run -d \
  --name fde-project-2 \
  --restart unless-stopped \
  --env-file .env \
  -p 8501:8501 \
  fde-project-2:latest
docker ps
docker logs -f fde-project-2
```

Then visit:

```text
http://YOUR_EC2_PUBLIC_IP:8501
```

---

## Final Deployment Flow

```text
GitHub
   |
   v
git clone
   |
   v
AWS EC2 Ubuntu
   |
   v
Docker Build
   |
   v
Docker Container
   |
   +--> FastAPI :8000
   |
   +--> Streamlit :8501
              |
              v
       Public Browser Access
```
