# HPC Services Setup Guide

Faculty of Mathematical Sciences — Ferdowsi University of Mashhad

Complete instructions for setting up all Docker-based services on a new system or restoring after Windows restart.

---

## Table of Contents

1. [System Requirements](#1-system-requirements)
2. [Prerequisites](#2-prerequisites)
3. [Service Overview](#3-service-overview)
4. [Setup Steps](#4-setup-steps)
   - [4.1 WSL2](#41-wsl2)
   - [4.2 Docker Desktop](#42-docker-desktop)
   - [4.3 Ollama](#43-ollama)
   - [4.4 LDAP Connection](#44-ldap-connection)
   - [4.5 LDAP Proxy](#45-ldap-proxy)
   - [4.6 JupyterHub](#46-jupyterhub)
   - [4.7 code-server (Lean)](#47-code-server-lean)
   - [4.8 Open WebUI](#48-open-webui)
   - [4.9 Python Shiny](#49-python-shiny)
   - [4.10 Whisper Transcription](#410-whisper-transcription)
5. [Auto-Start on Boot](#5-auto-start-on-boot)
6. [Backup and Restore](#6-backup-and-restore)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS | Windows 10/11 or Windows Server 2022 | Windows 11 / Server 2025 |
| CPU | 4 cores | 8+ cores |
| RAM | 32 GB | 64 GB |
| GPU | NVIDIA RTX 3060 (12 GB) | NVIDIA RTX 4090 (24 GB) |
| Storage (C:) | SSD 256 GB | SSD 512 GB |
| Storage (D:) | HDD/SSD 200 GB | SSD 500 GB |
| Network | University network or VPN | Tailscale for remote access |

---

## 2. Prerequisites

### Software to Install

1. **WSL2** with Ubuntu 24.04
2. **Docker Desktop** (with WSL2 backend)
3. **Ollama** (Windows native)
4. **Tailscale** (for remote access)

### Mirror Sources (for faster downloads in Iran)

| Package Manager | Mirror URL |
|----------------|------------|
| pip (Python) | `https://pypi.tuna.tsinghua.edu.cn/simple` |
| CRAN (R) | `https://cran.um.ac.ir` |
| Docker images | `docker.devneeds.ir` or `docker.arvancloud.ir` |
| Ollama | `https://cdn12.git.ir/softwares/ollama/ollama-linux-amd64.tar.zst` |

---

## 3. Service Overview

| Service | Port | Image | Description |
|---------|------|-------|-------------|
| JupyterHub | 8000 | `jupyterhub:complete` | Python/R notebooks with GPU |
| code-server | 31415 | `code-server-lean:mathlib` | Lean theorem prover IDE |
| Open WebUI | 3000 | `open-webui:latest` | LLM chat interface |
| Python Shiny | 8050 | `shiny-server-shiny` | Interactive web apps |
| Whisper | 8501 | `whisper-summarizer:fastapi` | Speech-to-text with LDAP |
| Ollama API | 11434 | (native) | LLM inference API |
| LDAP Proxy | 1389 | (Python script) | Forwards LDAP to AD server |

### Network Access

| Service | On-campus | Off-campus (Tailscale) |
|---------|-----------|------------------------|
| JupyterHub | `http://172.21.129.16:8000` | `http://math-gpu-server.tailde35cf.ts.net:8000` |
| code-server | `http://172.21.129.16:31415` | `http://math-gpu-server.tailde35cf.ts.net:31415` |
| Open WebUI | `http://172.21.129.16:3000` | `http://math-gpu-server.tailde35cf.ts.net:3000` |
| Python Shiny | `http://172.21.129.16:8050` | `http://math-gpu-server.tailde35cf.ts.net:8050` |
| Whisper | `http://172.21.129.16:8501` | `http://math-gpu-server.tailde35cf.ts.net:8501` |
| Ollama API | `http://172.21.129.16:11434` | `http://math-gpu-server.tailde35cf.ts.net:11434` |

---

## 4. Setup Steps

### 4.1 WSL2

```powershell
# In PowerShell (Admin)
wsl --install -d Ubuntu

# After restart, set WSL2 as default
wsl --set-default-version 2
```

Configure WSL memory in `C:\Users\<username>\.wslconfig`:

```ini
[wsl2]
memory=24GB
processors=6
swap=16GB
swapFile=D:\\temp\\wsl-swap.vhdx
localhostForwarding=true
```

Update Ubuntu:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential cmake pkg-config
```

### 4.2 Docker Desktop

1. Install Docker Desktop from [docker.com](https://www.docker.com/products/docker-desktop/)
2. Select "Use WSL 2 instead of Hyper-V"
3. Enable WSL2 integration in Settings → Resources → WSL Integration

Move Docker data to D: drive (optional):

```powershell
# In PowerShell (Admin)
mkdir D:\DockerData\DockerDesktopWSL
# Edit: C:\Users\<username>\AppData\Roaming\Docker\settings-store.json
# Add: "CustomWslDistroDir": "D:\\DockerData\\DockerDesktopWSL"
```

### 4.3 Ollama

Install from [ollama.com](https://ollama.com/download) or use mirror:

```
https://cdn12.git.ir/softwares/ollama/ollama-windows-amd64.exe
```

Download models:

```bash
ollama pull qwen3:8b
ollama pull qwen3:14b
ollama pull gemma3:4b
ollama pull llama3.2
```

Set model storage to D: drive (optional):

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_MODELS", "D:\ollama\models", "User")
```

### 4.4 LDAP Connection

The university uses Microsoft Active Directory (AD) for authentication. All services (JupyterHub, Whisper) authenticate users against this AD server.

#### University AD Server Details

| Parameter | Value |
|-----------|-------|
| **Server IP** | `172.20.9.29` |
| **Port** | `389` (LDAP) |
| **Domain** | `SAD` |
| **Base DN** | `DC=SAD,DC=UM,DC=AC,DC=IR` |
| **Bind DN Template** | `SAD\{username}` |

#### How LDAP Authentication Works

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   User Browser  │ ───> │  Docker Container│ ───> │  LDAP Proxy     │
│   (JupyterHub)  │      │  (JupyterHub)    │      │  (WSL Host)     │
└─────────────────┘      └─────────────────┘      └─────────────────┘
                                                           │
                                                           ▼
                                                   ┌─────────────────┐
                                                   │  University AD  │
                                                   │  172.20.9.29:389│
                                                   └─────────────────┘
```

**Why a proxy is needed:**
- Docker Desktop containers cannot directly reach the university's internal LDAP server
- The LDAP proxy runs on the WSL host and forwards LDAP traffic
- Containers connect to the proxy using the WSL host IP

#### Finding the WSL Host IP

The WSL host IP changes after each restart. Find it with:

```bash
# In WSL:
hostname -I | awk '{print $1}'

# Or check the Docker gateway IP:
docker network inspect bridge | grep Gateway
```

Common WSL host IP: `172.28.148.211` (may vary)

#### Testing LDAP Connectivity

```bash
# Test from WSL (direct connection to university AD):
ldapsearch -x -H ldap://172.20.9.29 -b "DC=SAD,DC=UM,DC=AC,DC=IR" -D "SAD\username" -W "(sAMAccountName=username)"

# Test via proxy (after starting LDAP proxy):
ldapsearch -x -H ldap://localhost:1389 -b "DC=SAD,DC=UM,DC=AC,DC=IR" -D "SAD\username" -W "(sAMAccountName=username)"
```

#### LDAP Configuration for Services

**JupyterHub Configuration:**
```python
c.LDAPAuthenticator.server_address = '172.28.148.211'  # WSL host IP
c.LDAPAuthenticator.server_port = 1389                  # LDAP proxy port
c.LDAPAuthenticator.bind_dn_template = 'SAD\\{username}'
c.LDAPAuthenticator.user_search_base = 'DC=SAD,DC=UM,DC=AC,DC=IR'
```

**Whisper Container Environment Variables:**
```bash
LDAP_SERVER=172.28.148.211  # WSL host IP
LDAP_PORT=1389              # LDAP proxy port
```

### 4.5 LDAP Proxy

The LDAP proxy is required because Docker Desktop containers cannot reach the university's internal LDAP server directly. The proxy runs on the WSL host and forwards LDAP traffic.

Create the proxy script:

```bash
cat > ~/jupyterlab-gpu/ldap_proxy.py << 'PYEOF'
#!/usr/bin/env python3
"""TCP proxy: forwards LDAP traffic from WSL to university AD server."""
import socket
import threading

LDAP_SERVER = '172.20.9.29'  # University AD server
LDAP_PORT = 389
LISTEN_PORT = 1389

def forward(src, dst):
    try:
        while True:
            data = src.recv(4096)
            if not data:
                break
            dst.sendall(data)
    except:
        pass
    finally:
        src.close()
        dst.close()

def handle_client(client_sock):
    try:
        server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        server_sock.connect((LDAP_SERVER, LDAP_PORT))
        t1 = threading.Thread(target=forward, args=(client_sock, server_sock))
        t2 = threading.Thread(target=forward, args=(server_sock, client_sock))
        t1.start()
        t2.start()
        t1.join()
        t2.join()
    except Exception as e:
        print(f'Proxy error: {e}')
        client_sock.close()

def main():
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(('0.0.0.0', LISTEN_PORT))
    sock.listen(5)
    print(f'LDAP proxy: {LISTEN_PORT} -> {LDAP_SERVER}:{LDAP_PORT}')
    while True:
        client, addr = sock.accept()
        threading.Thread(target=handle_client, args=(client,), daemon=True).start()

if __name__ == '__main__':
    main()
PYEOF
chmod +x ~/jupyterlab-gpu/ldap_proxy.py
```

Start the proxy:

```bash
nohup python3 ~/jupyterlab-gpu/ldap_proxy.py > /tmp/ldap_proxy.log 2>&1 &
```

### 4.6 JupyterHub

#### Option A: From backup image

```bash
gunzip -c /mnt/e/HPC-Backup/jupyterhub-complete.tar.gz | docker load
```

#### Option B: Build from Dockerfile

```bash
cd ~/jupyterlab-gpu
docker build -t jupyterhub:complete -f Dockerfile .
```

#### JupyterHub Configuration

The JupyterHub config is baked into the Docker image at `/etc/jupyterhub/jupyterhub_config.py`. Key settings:

```python
from ldapauthenticator import LDAPAuthenticator
from jupyterhub.spawner import SimpleLocalProcessSpawner

c.JupyterHub.authenticator_class = LDAPAuthenticator
c.LDAPAuthenticator.server_address = '172.28.148.211'  # WSL host IP (for proxy)
c.LDAPAuthenticator.server_port = 1389                  # LDAP proxy port
c.LDAPAuthenticator.bind_dn_template = 'SAD\\{username}'
c.LDAPAuthenticator.user_search_base = 'DC=SAD,DC=UM,DC=AC,DC=IR'

c.JupyterHub.spawner_class = SimpleLocalProcessSpawner
c.Authenticator.admin_users = {'m.amintoosi'}
c.Authenticator.allow_all = True
```

#### Start JupyterHub

```bash
cd ~/jupyterlab-gpu
docker run -d --name jupyterhub --gpus all -p 8000:8000 \
  -v ~/jupyterlab-gpu/work:/workspace \
  -v /mnt/d/DockerBackup:/datasets:ro \
  --env-file ~/jupyterlab-gpu/.env \
  jupyterhub:complete sleep infinity

sleep 5
docker exec -d jupyterhub bash -c 'cd /workspace && jupyterhub -f /etc/jupyterhub/jupyterhub_config.py'
```

### 4.7 code-server (Lean)

#### Restore from backup

```bash
gunzip -c /mnt/e/HPC-Backup/code-server-lean-mathlib.tar.gz | docker load
```

#### Start code-server

```bash
docker run -d \
  --name code-server \
  --gpus all \
  -p 31415:8080 \
  -v /home/amin/code-server/workspace:/home/coder/project \
  -v ~/.cache/huggingface:/home/coder/.cache/huggingface:ro \
  -v ~/.cache/torch:/home/coder/.cache/torch:ro \
  -e PASSWORD="fumlean" \
  -e DEFAULT_WORKSPACE=/home/coder/project \
  -e USER="coder" \
  --restart unless-stopped \
  code-server-lean:mathlib
```

**Note:** The workspace at `/home/amin/code-server/workspace/` must contain the Lean project files (`lean_tutorial/`, `mathlib4/`, etc.).

### 4.8 Open WebUI

```bash
docker run -d \
  --name open-webui \
  -p 3000:8080 \
  -e OLLAMA_BASE_URL=http://host.docker.internal:11434 \
  -v open-webui:/app/backend/data \
  --restart unless-stopped \
  docker.devneeds.ir/openwebui/open-webui:latest
```

Login: Register with your email, then admin approves at `m.amintoosi@um.ac.ir`.

### 4.9 Python Shiny

```bash
docker run -d \
  --name shiny-server \
  -p 8050:8050 \
  -v /home/amin/shiny-server/app:/app/app \
  -v /mnt/d/shiny-data:/app/data \
  --restart unless-stopped \
  shiny-server-shiny
```

### 4.10 Whisper Transcription

```bash
# Install ldap3 in container (required for LDAP auth)
docker run -d --name whisper-summarizer --gpus all -p 8501:8501 \
  -v /mnt/d/whisper-models/faster-whisper-medium:/models/faster-whisper-medium:ro \
  -v /mnt/e/whisper-models/faster-whisper-large-v3:/models/faster-whisper-large-v3:ro \
  -v /mnt/e/MAT/S2T:/mnt/e/MAT/S2T:rw \
  -e WHISPER_MODEL=/models/faster-whisper-medium \
  -e OLLAMA_HOST=http://172.21.129.16:11434 \
  -e OLLAMA_MODEL=qwen3:8b \
  -e LDAP_SERVER=172.28.148.211 \
  -e LDAP_PORT=1389 \
  --restart unless-stopped \
  whisper-summarizer:fastapi

# Install ldap3 and start server
docker exec whisper-summarizer pip3 install --break-system-packages ldap3
docker exec -d whisper-summarizer python3 /app/api.py
```

**Note:** The Whisper API file (`/app/api.py`) must include LDAP authentication. Use the version from `/home/amin/jupyterlab-gpu/whisper_api_auth.py`.

---

## 5. Auto-Start on Boot

### PowerShell Startup Script

Location: `D:\Start-AllServices.ps1`

This script:
1. Checks Docker Desktop health, performs full reset if needed
2. Starts Ollama
3. Starts JupyterHub (via shell script)
4. Starts Open WebUI
5. Starts Python Shiny
6. Starts Whisper
7. Starts code-server

### Batch File for Windows Startup

Location: `C:\Users\<username>\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup\Start-AllServices.bat`

```batch
@echo off
start /min powershell.exe -ExecutionPolicy Bypass -WindowStyle Hidden -File "D:\Start-AllServices.ps1"
```

### Startup Script (start-jupyterhub.sh)

Location: `~/jupyterlab-gpu/start-jupyterhub.sh`

This script:
1. Starts LDAP proxy if not running
2. Starts JupyterHub container
3. Copies updated config if available
4. Starts JupyterHub process

### code-server Restore Script

Location: `~/restore-code-server.sh`

This script is called by `Start-AllServices.ps1` and handles:
1. Checking if code-server container exists and is running
2. Verifying volume mounts are working (not tmpfs)
3. Recreating container if mounts are broken
4. Retrying once if first attempt fails

**Key features:**
- Automatically detects broken bind mounts
- Verifies files are visible in the workspace
- Retries once if Docker Desktop WSL2 integration is slow
- Provides clear error messages if manual intervention needed

---

## 6. Backup and Restore

### Backup Locations

| Item | Backup Location | Size |
|------|-----------------|------|
| JupyterHub image | `/mnt/e/HPC-Backup/jupyterhub-complete.tar.gz` | ~5 GB |
| code-server image | `/mnt/e/HPC-Backup/code-server-lean-mathlib.tar.gz` | ~1.3 GB |
| Open WebUI image | `/mnt/e/HPC-Backup/open-webui.tar.gz` | ~1.6 GB |
| Whisper image | `/mnt/e/HPC-Backup/whisper-summarizer.tar.gz` | ~7.3 GB |
| Shiny image | `/mnt/e/HPC-Backup/shiny-server.tar.gz` | ~0.7 GB |
| Ollama models | `/mnt/e/ollama/models` (separate copy) | ~100 GB |
| code-server workspace | `/home/amin/code-server/workspace/` | ~5 GB |
| JupyterHub workspace | `/home/amin/jupyterlab-gpu/work/` | varies |
| PowerShell startup script | `/mnt/e/HPC-Backup/Start-AllServices.ps1.bak` | ~9 KB |
| Batch startup file | `/mnt/e/HPC-Backup/Start-AllServices.bat.bak` | ~0.2 KB |
| JupyterHub startup script | `/mnt/e/HPC-Backup/start-jupyterhub.sh.bak` | ~1 KB |
| code-server restore script | `/mnt/e/HPC-Backup/restore-code-server.sh.bak` | ~3.7 KB |
| LDAP proxy script | `/mnt/e/HPC-Backup/ldap_proxy.py.bak` | ~1.3 KB |
| Whisper API script | `/mnt/e/HPC-Backup/whisper_api_auth.py.bak` | ~15 KB |
| JupyterHub env file | `/mnt/e/HPC-Backup/jupyterhub-env.bak` | ~0.1 KB |
| Docker Compose config | `/mnt/e/HPC-Backup/docker-compose.yml.bak` | ~0.8 KB |

### Backup Commands

```bash
# Run the automated backup script
bash /mnt/e/HPC-Backup/backup-all.sh
```

### Restore Commands

```bash
# Run the automated restore script
bash /mnt/e/HPC-Backup/restore-all.sh
```

---

## 7. Troubleshooting

### Docker Desktop fails to start

```powershell
# Full reset
taskkill /f /im "Docker Desktop.exe"
taskkill /f /im "com.docker.backend.exe"
wsl --shutdown
Stop-Service -Name "com.docker.service" -Force
Start-Sleep -Seconds 5
Start-Service -Name "com.docker.service"
Start-Process "C:\Program Files\Docker\Docker\Docker Desktop.exe"
```

### JupyterHub LDAP 500 error

1. Check LDAP proxy is running: `ps aux | grep ldap_proxy`
2. Restart proxy: `nohup python3 ~/jupyterlab-gpu/ldap_proxy.py > /tmp/ldap_proxy.log 2>&1 &`
3. Restart JupyterHub: `docker restart jupyterhub`

### code-server shows empty folders

**Root cause:** After Docker Desktop restarts, bind mounts from the WSL2 filesystem can silently resolve to tmpfs, making files invisible.

**Solution:** The `restore-code-server.sh` script handles this automatically:

```bash
# The script checks if mount is broken (tmpfs or empty workspace)
# If broken, it recreates the container with correct mounts
# If still broken after first attempt, it retries once
~/restore-code-server.sh
```

**Manual fix if needed:**

```bash
docker stop code-server && docker rm code-server
# Re-run with correct -v path
~/restore-code-server.sh
```

**Key points:**
- The script verifies the mount is working by checking files are visible
- It retries once if the first attempt fails (Docker Desktop WSL2 integration may need time)
- If still broken, try: `wsl --shutdown`, restart Docker Desktop, then re-run the script

### Whisper login fails

1. Check LDAP proxy: `ps aux | grep ldap_proxy`
2. Check environment variables: `docker inspect whisper-summarizer --format '{{.Config.Env}}'`
3. Verify LDAP_SERVER and LDAP_PORT are set correctly
4. Restart container with correct env vars

### Check container logs

```bash
docker logs <container-name> --tail 50
docker logs -f <container-name>  # follow logs
```

### Useful commands

```bash
docker ps                          # List running containers
docker ps -a                       # List all containers
docker images                      # List images
docker system df                   # Check disk usage
docker system prune -a             # Clean unused images
docker exec -it <container> bash   # Enter container
```

---

## File Structure

```
D:\
├── Start-AllServices.ps1          # Unified startup script
├── setup\
│   ├── Restart-JupyterHub.ps1     # Legacy JupyterHub starter
│   └── Restore-CodeServer.ps1     # Legacy code-server starter
└── ollama\models\                 # Ollama model files

E:\
├── MAT\
│   └── hpc\
│       ├── index.html             # User guide (Persian)
│       ├── install-manual.html    # Admin guide (Persian)
│       └── SETUP-GUIDE.md         # This file
├── HPC-Backup\
│   ├── backup-all.sh              # Automated backup script
│   ├── restore-all.sh             # Automated restore script
│   ├── *.tar.gz                   # Docker image backups
│   └── *.bak                      # Configuration backups
│       ├── Start-AllServices.ps1.bak
│       ├── Start-AllServices.bat.bak
│       ├── start-jupyterhub.sh.bak
│       ├── restore-code-server.sh.bak
│       ├── ldap_proxy.py.bak
│       ├── whisper_api_auth.py.bak
│       ├── jupyterhub-env.bak
│       └── docker-compose.yml.bak
├── ollama\models\                 # Ollama models (primary copy)
└── whisper-models\                # Whisper model files

/home/amin\
├── jupyterlab-gpu\
│   ├── start-jupyterhub.sh        # JupyterHub startup script
│   ├── ldap_proxy.py              # LDAP proxy script
│   ├── whisper_api_auth.py        # Whisper API with LDAP
│   ├── .env                       # Environment variables
│   ├── docker-compose.yml         # Docker Compose config
│   ├── Dockerfile                 # JupyterHub image build
│   ├── hub\
│   │   └── jupyterhub_config.py   # JupyterHub config (persistent)
│   └── work\                      # JupyterHub user data
├── restore-code-server.sh         # code-server restore/fix script
├── code-server\
│   ├── workspace\                 # Lean project files
│   └── Dockerfile.lean            # code-server image build
└── shiny-server\
    └── app\                       # Shiny app files
```

---

## Quick Reference

### Service URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| JupyterHub | http://localhost:8000 | University AD |
| code-server | http://localhost:31415 | Password: `fumlean` |
| Open WebUI | http://localhost:3000 | Guest: `guest@um.ac.ir` / `guest` |
| Python Shiny | http://localhost:8050 | — |
| Whisper | http://localhost:8501 | University AD |
| Ollama API | http://localhost:11434 | — |

### Contact

- Email: `m.amintoosi@um.ac.ir`
- Bale: `@mamintoosi`

---

*Last updated: July 12, 2026*

---

## Appendix: Backup Procedure

### When to Backup

- Before major system changes (Windows updates, Docker updates)
- After making configuration changes
- Periodically (weekly recommended)
- Before reinstalling services

### Running Backup

```bash
# Run the automated backup script
bash /mnt/e/HPC-Backup/backup-all.sh
```

This script backs up:
1. Docker images (JupyterHub, code-server, Open WebUI, Whisper, Shiny)
2. Configuration files (startup scripts, environment files, etc.)
3. PowerShell and batch startup files

### Backup Verification

After running backup, verify files exist:

```bash
ls -lh /mnt/e/HPC-Backup/*.tar.gz
ls -lh /mnt/e/HPC-Backup/*.bak
```

### Restoring from Backup

```bash
# Run the automated restore script
bash /mnt/e/HPC-Backup/restore-all.sh
```

This script restores:
1. Docker images from tar.gz files
2. Configuration files to their original locations
3. Startup scripts to Windows and WSL locations
