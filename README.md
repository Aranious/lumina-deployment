# Lumina Deployment (No‑Docker)

This repository contains the configuration files and step‑by‑step instructions to deploy the Lumina Django + Next.js application **directly on a Linux server** (without Docker).

---

## 1. Prerequisites

- Python 3.11 + venv
- Node.js 20 + pnpm
- Nginx
- PM2 (`npm install -g pm2`)
- Redis (optional, for Celery tasks)

---

## 2. Backend (Django + Gunicorn)

### Install dependencies
```bash
cd /path/to/backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt gunicorn
