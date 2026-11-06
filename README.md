# Company Intelligence Platform
### Enterprise Project, Team, Employee & AI-Powered Management System

A centralized internal company intelligence web platform that unifies project tracking, team workloads, individual employee contributions, real-time notifications with Google Chat, and conversational AI insights powered by OpenRouter.

---

## 🏛 System Architecture & Master Blueprints

Complete technical specifications and multi-agent development blueprints are documented in [`docs/`](./docs/):

1. **[System Architecture Specification](./docs/SYSTEM_ARCHITECTURE.md)**: High-level topology, multi-agent development organization, technology stack, and security model.
2. **[Database Design & ERD](./docs/DATABASE_DESIGN.md)**: Complete PostgreSQL entity relationship diagram, schema definitions, indexing, and historical preservation strategy.
3. **[Redmine Integration & Data Mapping](./docs/REDMINE_DATA_MAPPING.md)**: API discovery protocol, data mapping matrices, watermark delta synchronization, and rate-limiting resilience.
4. **[Notification Rules & Google Chat Integration](./docs/NOTIFICATION_RULES.md)**: Severity tiers, rule triggers, escalation matrix, and Google Chat Cards v2 webhook architecture.
5. **[AI Agent & Grounded Retrieval](./docs/AI_AGENT_ARCHITECTURE.md)**: OpenRouter multi-model gateway, deterministic tool calling, RBAC-aware context retrieval, and zero-hallucination citations.
6. **[API Design & Specifications](./docs/API_DESIGN.md)**: Modular FastAPI REST API routing tree, response envelopes, and RBAC dependency guards.
7. **[Phased Implementation Roadmap](./docs/IMPLEMENTATION_PLAN.md)**: Phase-by-phase delivery plan governed by the 7 specialized engineering agents.

---

## 🤖 Multi-Agent Development Organization

Development is structured across 7 specialized engineering roles:
- **Main Orchestrator Agent**: Architecture governance, task planning, integration, and code review.
- **Frontend Agent**: Next.js 14+ App Router, Tailwind CSS, 360 Dashboards, and UI visualizations.
- **Backend Agent**: FastAPI REST endpoints, SQLAlchemy async models, JWT authentication, and RBAC.
- **Redmine Agent**: Redmine API discovery, watermark delta sync, and entity mapping.
- **Notification Agent**: Rule evaluation, escalation cascades, and Google Chat webhook cards.
- **AI Agent**: OpenRouter LLM gateway, deterministic Python tool catalog, and grounded source citations.
- **QA Agent**: Test automation (pytest/Vitest), security auditing, and acceptance verification.

---

## 🚀 Technology Stack
- **Frontend**: Next.js 15+ (App Router), React 19, TypeScript, Tailwind CSS, Lucide Icons, Recharts.
- **Backend**: Python 3.11+, FastAPI, SQLAlchemy 2.0 (asyncio), Pydantic v2, Uvicorn.
- **Database**: SQLite with `aiosqlite` (built-in, zero-admin) or PostgreSQL 16+.
- **Process Management**: Systemd (Backend) & PM2 (Frontend).
- **Web Server / Reverse Proxy**: Nginx with Let's Encrypt TLS (Certbot).
- **Integrations**: Redmine REST API, Gmail / SMTP Email Alerts, Google OAuth 2.0, OpenRouter AI API.

---

# 🌐 Contabo VPS (8 GB RAM) Hosting & Deployment Guide

This guide walks you through deploying the Company Intelligence Platform on a **Contabo Cloud VPS** with **8 GB RAM** running **Ubuntu 22.04 LTS or 24.04 LTS**.

```
                           Internet (HTTPS / Port 443)
                                       │
                                       ▼
                       ┌───────────────────────────────┐
                       │   Nginx Reverse Proxy + TLS   │
                       └───────────────┬───────────────┘
                                       │
                    ┌──────────────────┴──────────────────┐
                    ▼                                     ▼
        ┌───────────────────────┐             ┌───────────────────────┐
        │   Next.js Frontend    │             │    FastAPI Backend    │
        │   PM2 (Node 20 LTS)   │             │   Systemd (Python)    │
        │   Port: 3000          │             │   Port: 8000          │
        └───────────────────────┘             └───────────┬───────────┘
                                                          │
                                                          ▼
                                              ┌───────────────────────┐
                                              │   SQLite / Postgres   │
                                              │   Persistent Storage  │
                                              └───────────────────────┘
```

---

### Step 1: Initial Server Setup & Swap Configuration

Log in to your Contabo VPS via SSH as `root`:
```bash
ssh root@YOUR_VPS_IP
```

Update system packages and create a dedicated deploy user:
```bash
apt update && apt upgrade -y
apt install -y curl wget git build-essential ufw fail2ban software-properties-common

# Create a non-root administrative user
adduser deploy
usermod -aG sudo deploy

# Optional: copy SSH keys to the deploy user
rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy/
```

#### Configure a 4GB Swapfile (Recommended for 8GB RAM VPS)
Adding a 4GB swap space guarantees stability during `npm run build` and burst background synchronization:
```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# Make permanent
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# Optimize swap aggressiveness for 8GB RAM
sysctl vm.swappiness=15
echo 'vm.swappiness=15' >> /etc/sysctl.conf
```

#### Configure UFW Firewall
```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp    # SSH
ufw allow 80/tcp    # HTTP
ufw allow 443/tcp   # HTTPS
ufw --force enable
```

---

### Step 2: Install Runtime Environments

Switch to the `deploy` user:
```bash
su - deploy
```

#### 1. Install Node.js 20 LTS (NodeSource) & PM2
```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g pm2
```

#### 2. Install Python 3.11+ and Virtualenv
```bash
sudo apt install -y python3 python3-pip python3-venv
python3 --version  # Should be 3.10, 3.11 or 3.12
```

#### 3. Install Nginx & Certbot
```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

---

### Step 3: Clone and Setup Repository

Clone the project into `/var/www/intelligent_projects_manager`:
```bash
sudo mkdir -p /var/www/intelligent_projects_manager
sudo chown -R deploy:deploy /var/www/intelligent_projects_manager

git clone https://github.com/YOUR_ORGANIZATION/intelligent_projects_manager.git /var/www/intelligent_projects_manager
cd /var/www/intelligent_projects_manager
```

---

### Step 4: Configure Backend (FastAPI)

```bash
cd /var/www/intelligent_projects_manager/backend

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Upgrade pip and install dependencies
pip install --upgrade pip
pip install -r requirements.txt
```

#### Create the Backend `.env` File
Create `/var/www/intelligent_projects_manager/backend/.env`:
```bash
nano /var/www/intelligent_projects_manager/backend/.env
```

Paste and adjust with your production credentials:
```env
# Application Settings
PROJECT_NAME="Company Intelligence Platform"
API_V1_PREFIX="/api/v1"
SECRET_KEY="generate-a-strong-random-64-character-secret-key"
APP_BASE_URL="https://pm.yourdomain.com"

# Root Super Admin Credentials
ADMIN_PASSWORD="ChooseAStrongAdminPassword123!"
DEFAULT_ALERT_EMAIL="your-admin@company.com"
DEFAULT_ALERT_NAME="System Administrator"

# Database Configuration (Embedded SQLite with aiosqlite)
DATABASE_URL="sqlite+aiosqlite:///./company_intel.db"

# Redmine API Integration
REDMINE_URL="https://redmine.yourdomain.com"
REDMINE_API_KEY="your-redmine-api-key"
REDMINE_SYNC_INTERVAL_MINUTES=1

# SMTP Email Alerting (Gmail or Custom SMTP)
SMTP_HOST="smtp.gmail.com"
SMTP_PORT=587
SMTP_USE_STARTTLS=true
SMTP_USERNAME="your-email@company.com"
SMTP_PASSWORD="your-google-app-password"
SMTP_FROM_EMAIL="your-email@company.com"
SMTP_FROM_NAME="Company Intelligence Platform"
EMAIL_ALERT_SEVERITIES="URGENT,CRITICAL"

# OpenRouter AI Integration
OPENROUTER_API_KEY="sk-or-v1-your-openrouter-key"
OPENROUTER_DEFAULT_MODEL="nvidia/nemotron-3.5-super-vl:free"
OPENROUTER_FALLBACK_MODEL="nvidia/nemotron-3.5-lightning:free"

# Google OAuth 2.0 Sign-In
GOOGLE_CLIENT_ID="your-client-id.apps.googleusercontent.com"
GOOGLE_CLIENT_SECRET="your-client-secret"
```

> **Security Tip**: Generate a strong `SECRET_KEY` with:
> ```bash
> python3 -c "import secrets; print(secrets.token_urlsafe(48))"
> ```

---

### Step 5: Configure & Build Frontend (Next.js)

```bash
cd /var/www/intelligent_projects_manager/frontend
```

Create `/var/www/intelligent_projects_manager/frontend/.env.local`:
```bash
nano .env.local
```

Paste:
```env
NEXT_PUBLIC_API_URL="https://pm.yourdomain.com/api/v1"
NEXT_PUBLIC_GOOGLE_CLIENT_ID="your-client-id.apps.googleusercontent.com"
```

Install packages and build the Next.js production bundle:
```bash
npm install
npm run build
```

---

### Step 6: Configure System Daemons

#### 1. Backend Daemon (`systemd`)
Create the systemd service file:
```bash
sudo nano /etc/systemd/system/company-intel-backend.service
```

Paste the following:
```ini
[Unit]
Description=Company Intelligence Platform FastAPI Backend
After=network.target

[Service]
Type=simple
User=deploy
Group=deploy
WorkingDirectory=/var/www/intelligent_projects_manager/backend
Environment="PATH=/var/www/intelligent_projects_manager/backend/venv/bin:/usr/local/bin:/usr/bin"
EnvironmentFile=/var/www/intelligent_projects_manager/backend/.env
ExecStart=/var/www/intelligent_projects_manager/backend/venv/bin/uvicorn main:app --workers 4 --host 127.0.0.1 --port 8000
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

Reload systemd and start the backend:
```bash
sudo systemctl daemon-reload
sudo systemctl enable company-intel-backend
sudo systemctl start company-intel-backend
sudo systemctl status company-intel-backend
```

#### 2. Frontend Daemon (`PM2`)
Start the Next.js server with PM2:
```bash
cd /var/www/intelligent_projects_manager/frontend
pm2 start npm --name "company-intel-frontend" -- start -- -p 3000

# Configure PM2 to start automatically on system reboot
pm2 startup
# (Run the sudo env PATH... command that PM2 outputs if prompted)
pm2 save
```

Verify status:
```bash
pm2 status
```

---

### Step 7: Configure Nginx & SSL Certificate

Create an Nginx configuration block:
```bash
sudo nano /etc/nginx/sites-available/company_intel.conf
```

Paste the following configuration (replace `pm.yourdomain.com` with your domain):
```nginx
# Rate limiting zone for authentication endpoints
limit_req_zone $binary_remote_addr zone=auth_limit:10m rate=10r/m;

server {
    listen 80;
    server_name pm.yourdomain.com;

    client_max_body_size 50M;

    # Gzip Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml application/json;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Backend API Proxy (/api/v1 -> FastAPI on 8000)
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 60s;
        proxy_send_timeout 120s;
        proxy_read_timeout 120s;
    }

    # OpenAPI Documentation
    location /docs {
        proxy_pass http://127.0.0.1:8000/docs;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /openapi.json {
        proxy_pass http://127.0.0.1:8000/openapi.json;
        proxy_set_header Host $host;
    }

    # Frontend Reverse Proxy (/ -> Next.js on 3000)
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Next.js Static Cache Optimization
    location /_next/static {
        proxy_pass http://127.0.0.1:3000;
        proxy_cache_valid 200 365d;
        add_header Cache-Control "public, max-age=31536000, immutable";
    }
}
```

Enable the site and verify Nginx syntax:
```bash
sudo ln -s /etc/nginx/sites-available/company_intel.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

#### Install Free SSL with Let's Encrypt Certbot
```bash
sudo certbot --nginx -d pm.yourdomain.com --non-interactive --agree-tos -m your-email@company.com --redirect
```

Certbot will automatically configure HTTPS on port 443 and auto-renewal.

---

### Step 8: Update Google OAuth Console for Production

In your [Google Cloud Console](https://console.cloud.google.com/) under **APIs & Services > Credentials > OAuth 2.0 Client IDs**:

1. **Authorized JavaScript Origins**:
   - Add: `https://pm.yourdomain.com`
2. **Authorized Redirect URIs**:
   - Add: `https://pm.yourdomain.com`
   - Add: `https://pm.yourdomain.com/`

---

### Step 9: Automated Database Backups (Cron Job)

To uphold the **Historical Continuity Invariant**, set up a nightly automated backup of `company_intel.db`:

```bash
sudo mkdir -p /var/backups/company_intel
sudo chown deploy:deploy /var/backups/company_intel
```

Create `/var/www/intelligent_projects_manager/backup_db.sh`:
```bash
#!/bin/bash
BACKUP_DIR="/var/backups/company_intel"
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
DB_FILE="/var/www/intelligent_projects_manager/backend/company_intel.db"

if [ -f "$DB_FILE" ]; then
    sqlite3 "$DB_FILE" ".backup '$BACKUP_DIR/company_intel_$TIMESTAMP.db'"
    # Retain backups for 14 days
    find "$BACKUP_DIR" -type f -name "*.db" -mtime +14 -exec rm {} \;
    echo "Backup completed: company_intel_$TIMESTAMP.db"
fi
```

Make it executable and add to crontab:
```bash
chmod +x /var/www/intelligent_projects_manager/backup_db.sh
crontab -e
```

Add this line to run daily at 2:00 AM:
```cron
0 2 * * * /var/www/intelligent_projects_manager/backup_db.sh >> /var/log/db_backup.log 2>&1
```

---

### Step 10: Zero-Downtime Redeployment Script

To update the application in the future with new code releases, create `deploy.sh`:

```bash
nano /var/www/intelligent_projects_manager/deploy.sh
chmod +x /var/www/intelligent_projects_manager/deploy.sh
```

Paste:
```bash
#!/bin/bash
set -e

echo "=== Pulling latest changes ==="
cd /var/www/intelligent_projects_manager
git pull origin main

echo "=== Updating Backend ==="
cd backend
source venv/bin/activate
pip install -r requirements.txt
sudo systemctl restart company-intel-backend

echo "=== Updating Frontend ==="
cd ../frontend
npm install
npm run build
pm2 reload company-intel-frontend

echo "=== Deployment Successfully Completed ==="
```

---

### 📊 Useful Monitoring Commands

| Task | Command |
|---|---|
| View Backend Live Logs | `journalctl -u company-intel-backend -f` |
| View Frontend Live Logs | `pm2 logs company-intel-frontend` |
| PM2 Realtime Dashboard | `pm2 monit` |
| Restart Backend Service | `sudo systemctl restart company-intel-backend` |
| Restart Frontend Service | `pm2 restart company-intel-frontend` |
| Test Nginx Configuration | `sudo nginx -t` |
| Check RAM & CPU Usage | `htop` or `free -h` |
