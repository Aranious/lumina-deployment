# 🔐 Complete Guide: Enabling HTTPS for Local Development with `mkcert`, Nginx, and WSL

This guide documents a real‑world debugging journey where a local development stack (Django + Next.js + Nginx) running on WSL (Windows Subsystem for Linux) failed to serve HTTPS correctly. The browser showed `ERR_CONNECTION_REFUSED` or `Not Secure`, and OpenSSL tests failed with `Name or service not known` or no certificate. After systematic fixes, the site runs on `https://lumina.localhost` with full browser trust, including password saving and modern web API access.

---

## 📋 Table of Contents

1. [The Problem](#the-problem)
2. [Root Causes](#root-causes)
3. [Step‑by‑Step Fixes](#step-by-step-fixes)
   - [1. Install `mkcert` and generate certificates](#1-install-mkcert-and-generate-certificates)
   - [2. Configure Nginx to use the certificates](#2-configure-nginx-to-use-the-certificates)
   - [3. Add `lumina.localhost` to hosts file](#3-add-luminalocalhost-to-hosts-file)
   - [4. Trust the `mkcert` root CA on Windows](#4-trust-the-mkcert-root-ca-on-windows)
   - [5. Update Django settings for proxy SSL headers](#5-update-django-settings-for-proxy-ssl-headers)
   - [6. Test and verify](#6-test-and-verify)
4. [What Changed? Why credentials now autofill](#what-changed-why-credentials-now-autofill)
5. [New Possibilities – Secure Context Features](#new-possibilities--secure-context-features)
6. [Final Project Status](#final-project-status)

---

## The Problem

- Local domain `https://lumina.localhost` refused connection (`ERR_CONNECTION_REFUSED`) or showed `Not Secure`.
- OpenSSL test: `Name or service not known` or `Could not read certificate from <stdin>`.
- Browser did not save passwords, autofill, or allow geolocation / service workers.
- Even after generating certificates with `mkcert`, Nginx appeared to ignore them.

## Root Causes

| Symptom                                                          | Underlying Cause                                                                                                                 |
| ---------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `Name or service not known`                                      | `lumina.localhost` missing from `/etc/hosts` (WSL) and `C:\Windows\System32\drivers\etc\hosts` (Windows).                        |
| `openssl s_client` could not read certificate                    | Nginx not listening on port 443, or certificate files missing/wrong permissions.                                                 |
| `Not Secure` in Windows browser even after `openssl` verified OK | Windows did not trust the `mkcert` root Certificate Authority (CA). The browser uses Windows certificate store, not the WSL one. |
| Passwords not saved / autofilled                                 | Browsers block credential storage and autofill on insecure origins (HTTP or untrusted HTTPS).                                    |

---

## Step‑by‑Step Fixes

### 1. Install `mkcert` and generate certificates

```bash
# Install mkcert on WSL (Ubuntu/Debian)
sudo apt install libnss3-tools
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64
sudo cp mkcert-v*-linux-amd64 /usr/local/bin/mkcert

# Install the local CA
mkcert -install

# Generate certificate for custom domain + localhost
cd ~/projects/lumina
mkcert lumina.localhost localhost 127.0.0.1 ::1
```

This creates two files: `lumina.localhost+3.pem` and `lumina.localhost+3-key.pem` (number may vary).

### 2. Configure Nginx to use the certificates

Copy certificates to Nginx’s ssl directory:

```bash
sudo mkdir -p /etc/nginx/ssl
sudo cp ~/projects/lumina/lumina.localhost+3.pem /etc/nginx/ssl/
sudo cp ~/projects/lumina/lumina.localhost+3-key.pem /etc/nginx/ssl/
sudo chmod 644 /etc/nginx/ssl/*.pem
sudo chmod 600 /etc/nginx/ssl/*-key.pem
```

Edit Nginx site configuration (e.g. `/etc/nginx/sites-available/lumina.localhost`). Example:

```nginx
upstream django_backends {
    server 127.0.0.1:8000;
    server 127.0.0.1:8001;
}
upstream nextjs_backends {
    server 127.0.0.1:3000;
    server 127.0.0.1:3001;
}

server {
    listen 80;
    server_name lumina.localhost;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name lumina.localhost;

    ssl_certificate     /etc/nginx/ssl/lumina.localhost+3.pem;
    ssl_certificate_key /etc/nginx/ssl/lumina.localhost+3-key.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

...
}
```

Enable the site and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/lumina.localhost /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 3. Add `lumina.localhost` to hosts file

**In WSL (`/etc/hosts`)** – add:

```
127.0.0.1   lumina.localhost
```

**In Windows (`C:\Windows\System32\drivers\etc\hosts`)** – same line. Run Notepad as Administrator to edit.

Flush DNS caches:

```bash
# WSL
sudo systemd-resolve --flush-caches

# Windows (CMD as Admin)
ipconfig /flushdns
```

### 4. Trust the `mkcert` root CA on Windows

The root CA file is in WSL at `$(mkcert -CAROOT)/rootCA.pem`. Copy it to Windows Desktop:

```bash
cp /home/atraxinous/.local/share/mkcert/rootCA.pem /mnt/c/Users/YourUsername/Desktop/
```

Then on Windows, install it:

**Method A – Double‑click + Certificate Import Wizard**

- Double‑click `rootCA.pem` → “Install Certificate” → **Local Machine** → **Place in “Trusted Root Certification Authorities”**.

**Method B – Using `certlm.msc`**

- Win+R → `certlm.msc` → Trusted Root Certification Authorities → Certificates → All Tasks → Import → select the file → place in “Trusted Root Certification Authorities”.

Restart Chrome and clear HSTS / socket pools:

- `chrome://net-internals/#hsts` → Delete domain `lumina.localhost`
- `chrome://net-internals/#sockets` → Flush socket pools

### 5. Update Django settings for proxy SSL headers

Because Nginx terminates SSL and forwards plain HTTP to Django, Django must know the original request was HTTPS. Add to `settings.py`:

```python
USE_X_FORWARDED_HOST = True
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
CSRF_TRUSTED_ORIGINS = ['https://lumina.localhost']
ALLOWED_HOSTS = ['localhost', '127.0.0.1', 'lumina.localhost']
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
```

Restart Gunicorn instances.

### 6. Test and verify

```bash
# Test SSL connection
echo | openssl s_client -connect lumina.localhost:443 -servername lumina.localhost 2>/dev/null | openssl x509 -text -noout | grep -E "Subject:|DNS:"
```

Expected output includes `DNS:lumina.localhost, DNS:localhost, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1`.

Visit `https://lumina.localhost` in Chrome – green lock appears.

---

## What Changed? Why credentials now autofill

Once the browser accepts the connection as **secure** (green lock, valid certificate), it classifies the origin as a **trusted secure context**. This unlocks:

| Previously (Not Secure)             | Now (Secure)                          |
| ----------------------------------- | ------------------------------------- |
| Browser refuses to save passwords   | Passwords can be saved and autofilled |
| No service workers                  | Service workers allowed               |
| Geolocation blocked                 | `navigator.geolocation` works         |
| Clipboard read/write denied         | Clipboard API available               |
| WebAuthn / biometrics blocked       | Passwordless login possible           |
| LocalStorage unreliable             | Full persistent storage               |
| Cookies with `Secure` flag rejected | `Secure` cookies accepted             |

This behavior is defined by browser security standards (W3C Secure Contexts). The `localhost` exception does **not** apply to custom domains like `lumina.localhost` – they must have a valid certificate.

---

## New Possibilities – Secure Context Features

Because your site is now served over trusted HTTPS, you can implement:

- **Offline support** via Service Workers
- **Push notifications** (requires service workers + permission)
- **Biometric authentication** (WebAuthn)
- **Hardware integration** (WebUSB, WebBluetooth)
- **Background sync** for offline form submissions
- **Secure payment flows** (Payment Request API)
- **Device orientation** for immersive experiences

Your Django + Next.js stack can now safely use JWT tokens or session cookies with the `Secure` flag, preventing man‑in‑the‑middle attacks.

---

## Final Project Status

- **Domain:** `https://lumina.localhost`
- **Nginx:** Listens on port 80 (redirects to 443) and 443 (SSL termination)
- **Upstreams:** Two Gunicorn instances (8000,8001) and two Next.js instances (3000,3001)
- **Certificates:** Generated with `mkcert`, trusted by both WSL and Windows
- **Browser trust:** Green lock, password saving, autofill, all secure APIs enabled

> ✅ Your local development environment now behaves exactly like a production HTTPS site.

---

## Troubleshooting Checklist

| If you see                          | Check this                                                            |
| ----------------------------------- | --------------------------------------------------------------------- |
| `ERR_CONNECTION_REFUSED`            | Nginx running? `sudo systemctl status nginx`                          |
| `Name or service not known`         | `/etc/hosts` and Windows `hosts` file                                 |
| `Not Secure` but OpenSSL works      | Windows does not trust mkcert root CA – import `rootCA.pem`           |
| Passwords not saved even with HTTPS | Clear browser cache / HSTS settings                                   |
| Mixed content warnings              | Ensure all subresources (images, scripts) use HTTPS or relative paths |

---

## Credits

- [mkcert](https://github.com/FiloSottile/mkcert) – local HTTPS made simple.
- Nginx, Django, Next.js – the stack that runs `lumina`.

_This guide was written after a real‑world debugging session. It is intended to serve as a ready‑to‑use README for anyone facing similar issues on WSL/Windows._
