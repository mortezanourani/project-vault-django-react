<div align="center">

# ProjectVault (Django + React)

### Project-Based File Storage & Management System

A self-hosted, full-stack file management platform built with **Django**, **Django Ninja**, and **React**. Designed for media production teams and creative organizations requiring structured, permission-aware file storage across multiple projects.

[![Python](https://img.shields.io/badge/Python-3.11+-3776AB?style=flat&logo=python&logoColor=white)](https://python.org)
[![Django](https://img.shields.io/badge/Django-5.1-092E20?style=flat&logo=django&logoColor=white)](https://djangoproject.com)
[![Django Ninja](https://img.shields.io/badge/Django_Ninja-1.3-009688?style=flat)](https://django-ninja.dev)
[![React](https://img.shields.io/badge/React-18-61DAFB?style=flat&logo=react&logoColor=black)](https://reactjs.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-15+-336791?style=flat&logo=postgresql&logoColor=white)](https://postgresql.org)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat)](LICENSE)

**Developed by [Morteza Nourani](https://github.com/mortezanourani) for [Rastaar Media Studio]**

</div>

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Architecture](#architecture)
- [Directory Structure](#directory-structure)
- [Storage Structure](#storage-structure)
- [Role & Permission Matrix](#role--permission-matrix)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Environment Variables](#environment-variables)
- [Production Deployment](#production-deployment)
- [Git Push Deployment](#git-push-deployment)
- [API Reference](#api-reference)
- [Tech Stack](#tech-stack)
- [Security](#security)
- [License](#license)

---

## Overview

**ProjectVault** is a self-hosted web application that replaces traditional SMB/NFS shared drives with a structured, role-aware file management interface. It organizes files by project, enforces directory-level permissions based on user roles, and provides in-app media viewing, file conflict resolution, and team mention notifications — all without relying on third-party cloud storage.

Originally developed for **Rastaar Media Studio** to manage production assets across multiple concurrent projects, it is designed to be deployable on-premises with full control over storage infrastructure.

---

## Features

### File Management

- **Project-based organization** — every project maintains its own isolated storage namespace
- **Automatic date directories** — date-stamped folders created on first file upload, preventing empty directory creation
- **Conflict detection** — detects duplicate filenames before upload with rename or overwrite resolution
- **Subdirectory support** — users can create custom subdirectories within Assets and Shared directories
- **Soft deletion** — deleted files are moved to a `.trash` folder per project, not permanently removed
- **Global storage** — a shared storage space accessible to all authenticated users, independent of project membership

### Media Handling

- **In-app media viewer** — view images, videos, and audio files directly in the browser without downloading
- **Automatic thumbnail generation** — image thumbnails via Pillow; video thumbnails via FFmpeg
- **File type icons** — distinct visual indicators for documents, spreadsheets, archives, and unsupported types
- **Viewer navigation** — keyboard arrow keys and thumbnail strip for browsing multiple files

### Access Control

- **Six-role hierarchy** — Administrator, Manager, Director, Coordinator, Editor, User
- **Global roles** — Administrator and Manager permissions apply across all projects
- **Per-project roles** — Director, Coordinator, Editor, and User roles assigned independently per project
- **Directory-level enforcement** — Edit directory access restricted to specific roles at the API level
- **JWT authentication** — stateless access tokens (15 min) with rotating refresh tokens (7 days)

### Notifications

- **Mention system** — tag one or more team members during file upload
- **In-app notification center** — bell icon with unread badge, updated via polling
- **Read management** — mark individual notifications or all notifications as read

### Administration

- **Custom admin panel** — full user and project management without exposing Django's built-in admin
- **User lifecycle** — create, activate, and deactivate user accounts
- **Project lifecycle** — create, activate, and deactivate projects
- **Member assignment** — add or remove project members and update their roles

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Client Browser                       │
└──────────────────────────────┬──────────────────────────────┘
                               │ HTTP / port 80
┌──────────────────────────────▼──────────────────────────────┐
│                            Nginx                             │
│   /              →   React SPA (frontend/dist/)              │
│   /api/          →   Gunicorn (proxy_pass :8000)             │
│   /static/       →   Django collected static files           │
└──────────────────────────────┬──────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────┐
│               Gunicorn WSGI Server (4 workers)               │
│                   Django 5.1 + Django Ninja                  │
│                                                              │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐  │
│  │  users   │  │ projects  │  │   files   │  │notifica- │  │
│  │  auth    │  │membership │  │ thumbnail │  │  tions   │  │
│  └──────────┘  └───────────┘  └─────┬─────┘  └──────────┘  │
└────────────────────────────────────┬┼─────────────────────── ┘
                                     ││
              ┌──────────────────────┘│
              │                       │
┌─────────────▼────────┐  ┌───────────▼──────────────────────┐
│      PostgreSQL       │  │          File Storage             │
│  Users, Projects,     │  │  Local filesystem or SMB mount    │
│  Files, Permissions   │  │  /mnt/storage or STORAGE_ROOT     │
└──────────────────────┘  └──────────────────────────────────┘
```

---

## Directory Structure

```
project-vault-django-react/
├── core/                        # Django project settings, root URLs, API router
├── users/                       # Custom user model, JWT auth, permission helpers
├── projects/                    # Project and membership models, directory services
├── files/                       # File records, upload, download, thumbnail logic
│   └── global_api.py            # Global storage API endpoints
├── notifications/               # Mention model, notification endpoints
├── frontend/                    # Vite + React application
│   ├── src/
│   │   ├── api/                 # Axios API client and per-domain modules
│   │   ├── components/          # Reusable UI components
│   │   ├── context/             # React AuthContext (JWT state management)
│   │   └── pages/               # Login, Projects, FileBrowser, Admin, GlobalStorage
│   ├── vite.config.js
│   └── package.json
├── manage.py
├── requirements.txt
└── .env                         # Environment variables — never commit this file
```

---

## Storage Structure

```
STORAGE_ROOT/
├── _global/                          # Global storage accessible to all users
│   ├── subfolder/
│   └── .trash/
│
├── project_slug/                     # Created when project is added
│   ├── 2024-01-15/                   # Auto-created on first upload (date format)
│   ├── 2024-01-16/
│   │
│   ├── Edit/                         # Created when coordinator is assigned
│   │   ├── 2024-01-15/               # Independent date directories
│   │   └── 2024-01-20/
│   │
│   ├── Assets/                       # Created when coordinator is assigned
│   │   └── custom-subfolder/         # User-created subdirectories (one level)
│   │
│   ├── Shared/                       # Created when coordinator is assigned
│   │   └── custom-subfolder/
│   │
│   └── .trash/                       # Soft-deleted files
│
└── .thumbs/                          # Auto-generated thumbnails
    ├── {file_id}.jpg                 # Project file thumbnails
    └── g/
        └── {file_id}.jpg             # Global file thumbnails
```

---

## Role & Permission Matrix

| Role | Scope | Date Dirs | Edit Dir | Assets | Shared | Delete | Assign Roles | Assign Users |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Administrator** | Global | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ | ✅ | ✅ |
| **Manager** | Global | ✅ Full | ✅ Full | ✅ Full | ✅ Full | ✅ | ❌ | ✅ |
| **Director** | Per Project | ⬆️⬇️👁 | ⬆️⬇️👁 | ⬆️⬇️👁 | ⬆️⬇️🗑️ | ❌ | ❌ | ❌ |
| **Coordinator** | Per Project | ⬆️⬇️👁 | ⬆️⬇️👁 | ⬆️⬇️👁 | ⬆️⬇️🗑️ | ❌ | ❌ | ❌ |
| **Editor** | Per Project | ⬆️⬇️👁 | ⬆️⬇️👁 | ⬆️⬇️👁 | ⬆️⬇️🗑️ | ❌ | ❌ | ❌ |
| **User** | Per Project | ⬆️⬇️👁 | ❌ | ⬆️⬇️👁 | ⬆️⬇️🗑️ | ❌ | ❌ | ❌ |

> ⬆️ Upload · ⬇️ Download · 👁 View · 🗑️ Delete own uploads in Shared only

**Notes:**
- Administrator and Manager roles are **global** — they apply to all projects without explicit assignment
- All other roles are **per-project** — a user may hold different roles in different projects simultaneously
- Coordinator assignment to a project triggers creation of Edit, Assets, and Shared directories
- All project members can create subdirectories within Assets and Shared

---

## Prerequisites

The following must be installed on the target system before proceeding:

| Dependency | Minimum Version | Purpose |
|---|---|---|
| Python | 3.11 | Runtime |
| Node.js | 20 | Frontend build |
| PostgreSQL | 15 | Primary database |
| FFmpeg | Any | Video thumbnail generation |
| libmagic | Any | MIME type detection |

```bash
# Ubuntu / Debian
sudo apt install -y \
  python3 python3-pip python3-venv \
  nodejs npm \
  postgresql postgresql-contrib \
  ffmpeg \
  libmagic1
```

---

## Installation

### 1. Clone the Repository

```bash
git clone https://github.com/mortezanourani/project-vault-django-react.git
cd project-vault-django-react
```

### 2. Configure PostgreSQL

```bash
sudo -u postgres psql << EOF
CREATE DATABASE storagemanager_db;
CREATE USER storagemanager_user WITH PASSWORD 'your_secure_password';
GRANT ALL PRIVILEGES ON DATABASE storagemanager_db TO storagemanager_user;
ALTER DATABASE storagemanager_db OWNER TO storagemanager_user;
\q
EOF

# Required for PostgreSQL 15 and above
sudo -u postgres psql -d storagemanager_db << EOF
GRANT ALL ON SCHEMA public TO storagemanager_user;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO storagemanager_user;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO storagemanager_user;
ALTER USER storagemanager_user CREATEDB;
\q
EOF
```

### 3. Python Environment

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 4. Environment Configuration

```bash
cp .env.example .env
nano .env
```

Fill in all required values as described in the [Environment Variables](#environment-variables) section.

### 5. Django Setup

```bash
python manage.py migrate
python manage.py createsuperuser
python manage.py collectstatic --noinput
```

### 6. Promote Superuser to Administrator

```bash
python manage.py shell -c "
from users.models import User
u = User.objects.get(username='your_username')
u.is_administrator = True
u.is_manager = True
u.save()
print('User promoted successfully.')
"
```

### 7. Create Storage Directories

```bash
mkdir -p /path/to/storage/.thumbs/g
```

### 8. Frontend Setup

```bash
cd frontend
npm install
```

Create `frontend/.env`:

```env
VITE_API_URL=http://localhost:8000/api
```

### 9. Run Development Servers

```bash
# Terminal 1 — Backend
source venv/bin/activate
python manage.py runserver

# Terminal 2 — Frontend
cd frontend
npm run dev
```

The application will be available at `http://localhost:5173`.
The interactive API documentation is available at `http://localhost:8000/api/docs`.

---

## Environment Variables

| Variable | Required | Description | Example |
|---|:---:|---|---|
| `SECRET_KEY` | ✅ | Django secret key (min. 50 characters) | `python3 -c "import secrets; print(secrets.token_urlsafe(50))"` |
| `DEBUG` | ✅ | Debug mode — always `False` in production | `False` |
| `ALLOWED_HOSTS` | ✅ | Comma-separated list of allowed hostnames | `192.168.1.50,localhost` |
| `DB_NAME` | ✅ | PostgreSQL database name | `storagemanager_db` |
| `DB_USER` | ✅ | PostgreSQL username | `storagemanager_user` |
| `DB_PASSWORD` | ✅ | PostgreSQL password | `your_secure_password` |
| `DB_HOST` | ✅ | PostgreSQL host | `localhost` |
| `DB_PORT` | ✅ | PostgreSQL port | `5432` |
| `STORAGE_ROOT` | ✅ | Absolute path to the file storage directory | `/mnt/storage` |

> ⚠️ The `.env` file must never be committed to version control. It is excluded via `.gitignore`.

---

## Production Deployment

### Gunicorn Service

Create `/etc/systemd/system/storagemanager.service`:

```ini
[Unit]
Description=ProjectVault — Gunicorn WSGI Server
After=network.target postgresql.service
RequiresMountsFor=/mnt/storage

[Service]
User=www-data
Group=www-data
WorkingDirectory=/var/www/storagemanager
EnvironmentFile=/var/www/storagemanager/.env
ExecStart=/var/www/storagemanager/venv/bin/gunicorn \
    --workers 4 \
    --bind 127.0.0.1:8000 \
    --timeout 300 \
    --graceful-timeout 300 \
    --keep-alive 5 \
    core.wsgi:application
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=mixed
Restart=on-failure
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable storagemanager
sudo systemctl start storagemanager
```

### Nginx Configuration

```nginx
server {
    listen 80;
    server_name your-domain.com;

    client_max_body_size 1024M;
    client_body_timeout 300s;
    client_body_buffer_size 10M;
    proxy_request_buffering off;

    root /var/www/storagemanager/frontend/dist;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }

    location /admin/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
    }

    location /static/ {
        alias /var/www/storagemanager/staticfiles/;
        expires 30d;
        add_header Cache-Control "public, no-transform";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### Service Startup Order

On system reboot, services start in the following order automatically:

```
Network interface ready
    → PostgreSQL starts
        → SMB/NFS storage mount (if applicable)
            → Gunicorn starts (waits for PostgreSQL)
                → Nginx starts
                    → Application available
```

---

## Git Push Deployment

### One-Time Server Setup

```bash
# On the server
mkdir -p /var/repo/storagemanager.git
cd /var/repo/storagemanager.git
git init --bare

# Create post-receive hook
cat > hooks/post-receive << 'EOF'
#!/bin/bash
set -e

APP_DIR="/var/www/storagemanager"
VENV="$APP_DIR/venv"

echo "==> Deploying ProjectVault..."

git --work-tree=$APP_DIR --git-dir=/var/repo/storagemanager.git checkout -f main

cd $APP_DIR

echo "==> Installing Python dependencies..."
source $VENV/bin/activate
pip install -r requirements.txt --quiet

echo "==> Running database migrations..."
python manage.py migrate --noinput

echo "==> Collecting static files..."
python manage.py collectstatic --noinput

echo "==> Building React frontend..."
cd frontend
npm install --silent
npm run build

echo "==> Reloading application server..."
sudo systemctl reload storagemanager

echo "==> Deployment complete."
EOF

chmod +x hooks/post-receive
```

### Developer Workflow

```bash
# One-time remote setup (on developer machine)
git remote add production ssh://user@your-server/var/repo/storagemanager.git

# Deploy
git push production main
```

---

## API Reference

Full interactive documentation is available at `/api/docs` (Swagger UI) when the server is running.

| Tag | Method | Endpoint | Description | Auth Required |
|---|---|---|---|:---:|
| **Auth** | POST | `/api/auth/login` | Authenticate with username and password | ❌ |
| **Auth** | POST | `/api/auth/refresh` | Issue new access token from refresh token | ❌ |
| **Auth** | POST | `/api/auth/logout` | Invalidate session | ✅ |
| **Users** | GET | `/api/users/me` | Retrieve current user profile | ✅ |
| **Users** | GET | `/api/users` | List all users | ✅ Admin |
| **Users** | POST | `/api/users` | Create a new user | ✅ Admin |
| **Users** | PATCH | `/api/users/{id}` | Update user fields | ✅ Admin |
| **Users** | DELETE | `/api/users/{id}` | Deactivate a user | ✅ Admin |
| **Projects** | GET | `/api/projects` | List accessible projects | ✅ |
| **Projects** | POST | `/api/projects` | Create a project | ✅ Admin |
| **Projects** | GET | `/api/projects/{id}` | Retrieve project details | ✅ |
| **Projects** | PATCH | `/api/projects/{id}` | Update project | ✅ Admin |
| **Projects** | GET | `/api/projects/{id}/structure` | Get directory tree | ✅ |
| **Projects** | GET | `/api/projects/{id}/members` | List project members | ✅ |
| **Projects** | POST | `/api/projects/{id}/members` | Add project member | ✅ Manager |
| **Projects** | DELETE | `/api/projects/{id}/members/{uid}` | Remove project member | ✅ Manager |
| **Files** | GET | `/api/projects/{id}/files` | List files in a directory | ✅ |
| **Files** | POST | `/api/projects/{id}/files/check-conflict` | Check for filename conflict | ✅ |
| **Files** | POST | `/api/projects/{id}/files/upload` | Upload a file | ✅ |
| **Files** | GET | `/api/projects/{id}/files/{fid}/download` | Download a file | ✅ |
| **Files** | GET | `/api/projects/{id}/files/{fid}/thumbnail` | Retrieve thumbnail | ✅ |
| **Files** | DELETE | `/api/projects/{id}/files/{fid}` | Soft delete a file | ✅ Manager |
| **Files** | POST | `/api/projects/{id}/directories` | Create subdirectory | ✅ |
| **Global** | GET | `/api/global/structure` | Global storage directory tree | ✅ |
| **Global** | GET | `/api/global/files` | List global files | ✅ |
| **Global** | POST | `/api/global/files/upload` | Upload to global storage | ✅ |
| **Global** | GET | `/api/global/files/{id}/download` | Download global file | ✅ |
| **Global** | DELETE | `/api/global/files/{id}` | Delete global file | ✅ Owner/Manager |
| **Global** | POST | `/api/global/directories` | Create global subdirectory | ✅ |
| **Notifications** | GET | `/api/notifications` | List notifications | ✅ |
| **Notifications** | GET | `/api/notifications/count` | Get unread count | ✅ |
| **Notifications** | PATCH | `/api/notifications/{id}/read` | Mark one as read | ✅ |
| **Notifications** | PATCH | `/api/notifications/mark-all-read` | Mark all as read | ✅ |

---

## Tech Stack

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **Backend Framework** | Django | 5.1 | Web framework, ORM, migrations |
| **API Layer** | Django Ninja | 1.3 | REST API with Python type hints |
| **Authentication** | djangorestframework-simplejwt | 5.3 | JWT access and refresh tokens |
| **Database** | PostgreSQL | 15+ | Primary relational database |
| **Database Driver** | psycopg2-binary | 2.9 | Python-PostgreSQL adapter |
| **Frontend Framework** | React | 18 | Single-page application |
| **Frontend Build** | Vite | 5 | Development server and production build |
| **UI Component Library** | Ant Design | 5 | UI components |
| **HTTP Client** | Axios | 1.x | API communication with interceptors |
| **Client Routing** | React Router | v6 | Client-side navigation |
| **Image Processing** | Pillow | 10.4 | Image thumbnail generation |
| **Video Processing** | FFmpeg | — | Video thumbnail extraction |
| **MIME Detection** | python-magic | 0.4 | File type detection from content |
| **Configuration** | python-dotenv | 1.0 | Environment variable management |
| **CORS** | django-cors-headers | 4.4 | Cross-origin request handling |
| **WSGI Server** | Gunicorn | 22 | Production application server |
| **Reverse Proxy** | Nginx | — | Static files, SSL, request routing |

---

## Security

- **Access tokens** expire after 15 minutes; refresh tokens expire after 7 days with automatic rotation
- **All file access** is validated server-side against role and project membership before any file is served
- **MIME type verification** uses `python-magic` to inspect file content rather than trusting the filename extension
- **Password hashing** uses Django's default PBKDF2-SHA256 algorithm
- **Soft deletion** ensures files are moved to `.trash` and not immediately exposed or accessible
- **Environment variables** containing credentials are excluded from version control via `.gitignore`
- **Directory traversal** is prevented by resolving all paths relative to `STORAGE_ROOT`

---

## License

```
MIT License

Copyright (c) 2025 Morteza Nourani — Rastaar Media Studio

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

---

<div align="center">

Developed and maintained by **[Morteza Nourani](https://github.com/mortezanourani)**
for **Rastaar Media Studio**

</div>
