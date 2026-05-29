The root cause: `node_modules/.bin/next` is a shell script, not a Node.js file. PM2 tried to run it as JavaScript, causing the syntax error you saw. The fix is to point PM2 at the real entry point: `node_modules/next/dist/bin/next`.

I’ve written a complete, corrected `README.md` for your `lumina-deployment` repository. It includes:

- The fixed PM2 start commands.
- An optional ecosystem file for convenience.
- Clear steps for rebuilding and restarting after code changes.
- Instructions for the backend, Nginx, SSL, etc.

---

```markdown
# Lumina Deployment

Infrastructure and configuration files for deploying **Lumina** (Django + Next.js) on a Linux VPS **without Docker**.

---

## Overview

- **Reverse Proxy**: Nginx (with load balancing)
- **Backend**: Django + Gunicorn (2 instances on ports 8000, 8001)
- **Frontend**: Next.js 15 (2 production instances on ports 3000, 3001, managed by PM2)
- **Database**: PostgreSQL + pgvector
- **Task Queue**: Redis + Celery (configured in the main repo)
- **Process Manager**: systemd (Django & Celery), PM2 (Next.js)

---

## Folder Structure
```

lumina-deployment/
├── nginx/
│ └── lumina.conf
├── systemd/
│ ├── lumina-django.service
│ ├── lumina-django2.service
│ ├── celery-worker.service
│ └── celery-beat.service
├── scripts/
│ └── renew-ssl.sh
├── README.md
└── .env.example

````

---

## Prerequisites

- Ubuntu 22.04 / 24.04
- Python 3.11+
- Node.js 20+ and pnpm
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
````

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

# Production build
pnpm build
```

**After building, start the production servers with PM2:**

```bash
# IMPORTANT: Use the real Node.js entry point, not the shell script!
pm2 start node_modules/next/dist/bin/next --name "lumina-frontend" -- start

# Second instance on port 3001
pm2 start node_modules/next/dist/bin/next --name "lumina-frontend-2" -- start -p 3001

# Save process list and enable startup on boot
pm2 save
pm2 startup   # follow the on-screen instructions to install the startup hook
```

**Optional:** Use an ecosystem file for easier management. Create `ecosystem.config.js` in the frontend directory:

```javascript
module.exports = {
  apps: [
    {
      name: "lumina-frontend",
      script: "node_modules/next/dist/bin/next",
      args: "start",
      cwd: "/home/atraxinous/projects/lumina/frontend",
      env: {
        NODE_ENV: "production",
        PORT: 3000,
      },
    },
    {
      name: "lumina-frontend-2",
      script: "node_modules/next/dist/bin/next",
      args: "start",
      cwd: "/home/atraxinous/projects/lumina/frontend",
      env: {
        NODE_ENV: "production",
        PORT: 3001,
      },
    },
  ],
};
```

Then start both with:

```bash
pm2 start ecosystem.config.js --env production
pm2 save
```

### 4. Copy Configuration Files

```bash
# Nginx
sudo cp ../lumina-deployment/nginx/lumina.conf /etc/nginx/sites-available/lumina
sudo ln -sf /etc/nginx/sites-available/lumina /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx

# Systemd services (Django, Celery)
sudo cp ../lumina-deployment/systemd/lumina-*.service /etc/systemd/system/
sudo cp ../lumina-deployment/systemd/celery-*.service /etc/systemd/system/
sudo systemctl daemon-reload
```

### 5. Start Services

```bash
# Django (Gunicorn)
sudo systemctl enable --now lumina-django lumina-django2

# Celery
sudo systemctl enable --now celery-worker celery-beat

# Next.js is already running via PM2 (see step 3)
```

---

## Service Management

### Status

```bash
sudo systemctl status lumina-django
sudo systemctl status celery-worker
pm2 status
```

### Restart after code changes

**Backend (Django):**

```bash
cd ~/projects/lumina/backend
source venv/bin/activate
python manage.py migrate          # only if models changed
python manage.py collectstatic --noinput
sudo systemctl restart lumina-django lumina-django2
sudo systemctl restart celery-worker celery-beat
```

**Frontend (Next.js):**

```bash
cd ~/projects/lumina/frontend
pnpm build
pm2 restart all
```

### Logs

```bash
# Django
journalctl -u lumina-django -f

# Celery
journalctl -u celery-worker -f

# Next.js (PM2)
pm2 logs
```

---

## Nginx Configuration

Located at `nginx/lumina.conf`. It handles:

- HTTP → HTTPS redirect
- Load balancing between two Gunicorn instances (8000/8001)
- Load balancing between two Next.js instances (3000/3001)
- Serving `/static/` and `/media/` directly from the filesystem

---

## SSL Setup (Production)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

---

## Important Notes

- **The Next.js production server MUST be started after every build.**  
  PM2 is the recommended process manager. Never use `pnpm dev` in production.
- The `node_modules/.bin/next` file is a shell script – do **not** try to run it with PM2. Use `node_modules/next/dist/bin/next` instead (as shown above).
- Celery configuration lives in the main Lumina repository (`backend/lumina/celery.py`).
- Environment variables (e.g., `DATABASE_URL`, `REDIS_URL`, `EMAIL_HOST_PASSWORD`) should be set in the systemd service files or in a `.env` file loaded by Django.

---

## Related Repositories

- **[Lumina Main Application](https://github.com/Aranious/lumina)** – Source code, models, views, Celery config

```

Replace your current `README.md` in the `lumina-deployment` repo with this content. Then, any future deployment (or re‑deployment after pulling changes) will work correctly by following these instructions.
```
