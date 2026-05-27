# Lumina Deployment

This repository contains all infrastructure and configuration files for deploying **Lumina** (Django + Next.js) on a Linux VPS **without Docker**.

It keeps the main application code clean by separating deployment-specific files (Nginx, systemd services, etc.).

---

## Overview

- **Reverse Proxy**: Nginx (with load balancing)
- **Backend**: Django + Gunicorn (2 instances)
- **Frontend**: Next.js 15 (2 instances via PM2)
- **Database**: PostgreSQL + pgvector
- **Task Queue**: Redis + Celery (configured in main repo)
- **Process Manager**: systemd + PM2

---

## Folder Structure

```bash
lumina-deployment/
├── nginx/
│   └── lumina.conf
├── systemd/
│   ├── lumina-django.service
│   └── lumina-django2.service
├── scripts/
│   └── renew-ssl.sh
├── README.md
└── .env.example
```

---

## Prerequisites

- Ubuntu 22.04 / 24.04
- Python 3.11+
- Node.js 20+ + pnpm
- PostgreSQL 14+ with `pgvector` extension
- Redis
- Nginx
- PM2 (`npm install -g pm2`)

---

## Deployment Steps

### 1. Clone Repositories

```bash
cd /home/atraxinous/projects
git clone https://github.com/Aranious/lumina.git
git clone https://github.com/Aranious/lumina-deployment.git
```

### 2. Backend Setup (Django)

```bash
cd /home/atraxinous/projects/lumina/backend

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt gunicorn

python manage.py collectstatic --noinput
python manage.py migrate
```

### 3. Frontend Setup (Next.js)

```bash
cd /home/atraxinous/projects/lumina/frontend
pnpm install
pnpm build
```

### 4. Copy Configuration Files

```bash
# Nginx
sudo cp ../lumina-deployment/nginx/lumina.conf /etc/nginx/sites-available/lumina
sudo ln -s /etc/nginx/sites-available/lumina /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx

# Systemd services
sudo cp ../lumina-deployment/systemd/* /etc/systemd/system/
sudo systemctl daemon-reload
```

### 5. Start Services

```bash
# Django (Gunicorn)
sudo systemctl enable --now lumina-django lumina-django2

# Celery (Worker + Beat)
sudo systemctl enable --now celery-worker celery-beat

# Next.js (PM2)
cd /home/atraxinous/projects/lumina/frontend
pm2 start ecosystem.config.js --env production
pm2 save
```

---

## Service Management

```bash
# Status
sudo systemctl status lumina-django
sudo systemctl status lumina-django2
sudo systemctl status celery-worker
sudo systemctl status celery-beat
pm2 status

# Restart (after code changes)
sudo systemctl restart lumina-django lumina-django2
sudo systemctl restart celery-worker celery-beat
pm2 restart all

# Logs
journalctl -u lumina-django -f
journalctl -u celery-worker -f
pm2 logs
```

---

## Important Notes

- **Celery Configuration**: The `celery.py` file and task definitions are in the **main Lumina repository** (`backend/lumina/celery.py`). The systemd services in this repo only run the Celery worker and beat using that configuration.
- **Environment Variables**: Make sure `.env` or environment variables in the systemd service files include proper Redis and email settings.
- **Gunicorn**: Two instances (8000 & 8001) for better concurrency and resilience.
- **Next.js**: Served via PM2 for production process management.

---

## Nginx Configuration

Located at `nginx/lumina.conf`. It handles:

- HTTP → HTTPS redirect
- Load balancing between Django instances
- Load balancing between Next.js instances
- Efficient serving of `/static/` and `/media/`

---

## SSL Setup (Production)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d lumina.com
```

---

## Related Repositories

- **[Lumina Main Application](https://github.com/Aranious/lumina)** — Source code, models, views, and Celery config
