## Implementation

This implementation assumes **Clouding.io**, **one Ubuntu 24.04 VPS**, **Docker preinstalled image**, and a **single-server MVP**. Clouding currently offers a Docker image on Ubuntu 24.04, supports snapshots/backups and firewall management, and also supports private networks if you later split the database onto a second server. ([help.clouding.io][1])

### 1. Create the server

Create one Clouding server with:

* **Image**: Docker on Ubuntu 24.04
* **Public IP**: enabled
* **Firewall profile**: allow only SSH, HTTP, HTTPS
* **Backups**: enabled
* **Snapshot**: take one before first production deploy

Clouding documents the Docker preinstalled image on Ubuntu 24.04, server creation via the control panel, and the availability of firewall, backups, snapshots, and private networking. ([help.clouding.io][1])

### 2. Base server hardening

Immediately after SSH login:

* create a non-root sudo user
* install system updates
* disable password SSH auth if possible
* enable UFW
* set timezone to `Europe/Athens`
* create directories for app data and backups

Example:

```bash
sudo adduser garmin
sudo usermod -aG sudo garmin

sudo apt update && sudo apt upgrade -y
sudo timedatectl set-timezone Europe/Athens

sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

sudo mkdir -p /opt/garmin-stack/{app,postgres,imports,exports,backups,nginx}
sudo chown -R garmin:garmin /opt/garmin-stack
```

### 3. Repository and deployment layout

On the server, deploy into:

```text
/opt/garmin-stack/
  app/           # git checkout
  postgres/      # postgres data volume or bind mount
  imports/       # Garmin export files
  exports/       # generated HTML reports
  backups/       # pg_dump backups
  nginx/         # nginx config
```

Use this repo layout:

```text
repo/
  app/
    dashboard/
      streamlit_app.py
      pages/
    core/
      config.py
      db.py
      logging.py
    models/
    pipelines/
    services/
  migrations/
  scripts/
    run_pipeline.py
    bootstrap_from_sqlite.py
  deploy/
    docker-compose.yml
    nginx.conf
    .env.example
    systemd/
      garmin-pipeline.service
      garmin-pipeline.timer
      garmin-backup.service
      garmin-backup.timer
  Dockerfile
  requirements.txt
```

### 4. Dockerize the application

Use one application image for both the Streamlit app and the pipeline runner.

#### `Dockerfile`

```dockerfile
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential curl libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN useradd -ms /bin/bash appuser
USER appuser

CMD ["streamlit", "run", "app/dashboard/streamlit_app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

### 5. Define the Compose stack

Use **Docker Compose** with three containers:

* `postgres`
* `dashboard-web`
* `nginx`

The pipeline runs on demand through `docker compose run --rm pipeline-runner ...`, not as a permanently running container.

#### `deploy/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16
    container_name: garmin-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: garmin
      POSTGRES_USER: garmin
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - /opt/garmin-stack/postgres:/var/lib/postgresql/data
    networks:
      - internal

  dashboard-web:
    build:
      context: ..
      dockerfile: Dockerfile
    container_name: garmin-dashboard
    restart: unless-stopped
    env_file:
      - .env
    command: >
      streamlit run app/dashboard/streamlit_app.py
      --server.port=8501
      --server.address=0.0.0.0
    volumes:
      - /opt/garmin-stack/exports:/data/exports
      - /opt/garmin-stack/imports:/data/imports
    depends_on:
      - postgres
    networks:
      - internal

  nginx:
    image: nginx:1.27
    container_name: garmin-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /opt/garmin-stack/nginx/nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - /opt/garmin-stack/nginx/certs:/etc/letsencrypt
      - /opt/garmin-stack/nginx/www:/var/www/certbot
    depends_on:
      - dashboard-web
    networks:
      - internal

networks:
  internal:
    driver: bridge
```

### 6. Configure environment variables

Create `deploy/.env`:

```bash
APP_ENV=prod
TZ=Europe/Athens

DATABASE_URL=postgresql+psycopg://garmin:CHANGE_ME@postgres:5432/garmin
POSTGRES_PASSWORD=CHANGE_ME

ATHLETE_TIMEZONE=Europe/Athens
GARMIN_IMPORT_PATH=/data/imports
HTML_EXPORT_PATH=/data/exports

LOG_LEVEL=INFO
```

Use **Pydantic Settings** or equivalent to load these centrally in `app/core/config.py`.

### 7. Configure Nginx

Clouding’s Ubuntu guidance uses Nginx as the public entry point, which is also the right choice here for TLS termination and proxying to Streamlit. ([help.clouding.io][2])

#### `deploy/nginx.conf`

```nginx
server {
    listen 80;
    server_name your-domain.example;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    server_name your-domain.example;

    ssl_certificate /etc/letsencrypt/live/your-domain.example/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.example/privkey.pem;

    client_max_body_size 50M;

    location / {
        proxy_pass http://dashboard-web:8501;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 300;
    }
}
```

### 8. First boot of the stack

From `/opt/garmin-stack/app/deploy`:

```bash
docker compose up -d --build
```

Check:

```bash
docker compose ps
docker compose logs -f postgres
docker compose logs -f dashboard-web
docker compose logs -f nginx
```

At this point the dashboard should be reachable through Nginx once DNS and TLS are in place.

### 9. Add TLS

Point your domain DNS to the Clouding server public IP, then issue Let’s Encrypt certificates.

One simple pattern is a temporary certbot container or host-installed certbot. The important part is that only ports 80 and 443 are exposed publicly. Clouding’s platform supports public IP plus firewall control, so this is the correct exposure model. ([clouding.io][3])

### 10. Add database migrations

Use **Alembic** and run migrations as part of every deploy.

Initial sequence:

```bash
docker compose run --rm dashboard-web alembic upgrade head
```

For production, wrap deployment as:

```bash
git pull
docker compose build
docker compose run --rm dashboard-web alembic upgrade head
docker compose up -d
```

### 11. Bootstrap data from SQLite

Run a one-time migration from your existing local SQLite databases into PostgreSQL.

Example command:

```bash
docker compose run --rm \
  -v /opt/garmin-stack/imports:/data/imports \
  dashboard-web \
  python scripts/bootstrap_from_sqlite.py
```

Implementation rules for `bootstrap_from_sqlite.py`:

* open each SQLite DB read-only
* map source tables into canonical Postgres tables
* bulk insert in batches
* use upserts on unique business keys
* write a row into `import_batch`
* log counts and failures to `job_run`

This lets you keep your local SQLite files as the historical source while moving the online system fully to Postgres.

### 12. Implement the scheduled pipeline

On Clouding, the scheduled pipeline should be triggered by **systemd timer** or cron on the host. This replaces Railway cron jobs.

#### `deploy/systemd/garmin-pipeline.service`

```ini
[Unit]
Description=Garmin analytics pipeline

[Service]
Type=oneshot
WorkingDirectory=/opt/garmin-stack/app/deploy
ExecStart=/usr/bin/docker compose run --rm dashboard-web python scripts/run_pipeline.py
```

#### `deploy/systemd/garmin-pipeline.timer`

```ini
[Unit]
Description=Run Garmin analytics pipeline every 6 hours

[Timer]
OnBootSec=10min
OnUnitActiveSec=6h
Persistent=true

[Install]
WantedBy=timers.target
```

Install:

```bash
sudo cp deploy/systemd/garmin-pipeline.service /etc/systemd/system/
sudo cp deploy/systemd/garmin-pipeline.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now garmin-pipeline.timer
```

Check:

```bash
systemctl list-timers --all | grep garmin
journalctl -u garmin-pipeline.service -n 100 --no-pager
```

### 13. Add automated PostgreSQL backups

Clouding supports server backups and snapshots, but you should still add **application-level Postgres dumps** so you can restore individual databases cleanly. Clouding’s platform advertises backups and snapshots; use both platform-level protection and DB-level dumps. ([clouding.io][3])

#### `deploy/systemd/garmin-backup.service`

```ini
[Unit]
Description=Nightly PostgreSQL backup

[Service]
Type=oneshot
WorkingDirectory=/opt/garmin-stack/app/deploy
ExecStart=/bin/bash -lc '/usr/bin/docker exec garmin-postgres pg_dump -U garmin -d garmin > /opt/garmin-stack/backups/garmin_$(date +%%F).sql'
```

#### `deploy/systemd/garmin-backup.timer`

```ini
[Unit]
Description=Nightly PostgreSQL backup timer

[Timer]
OnCalendar=*-*-* 03:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

Then add retention, for example deleting dumps older than 14 days.

### 14. Logging and health checks

Implement logging in two places:

* application logs to stdout
* systemd logs for timers and host-level scheduling

Minimum health endpoints/checks:

* dashboard startup succeeds
* Postgres accepts connections
* pipeline last successful run timestamp
* last import batch status
* row counts per job

Expose a small “Pipeline Status” page in Streamlit reading from `job_run` and `import_batch`.

### 15. Deployment workflow

For the MVP, use this release flow:

1. develop locally
2. push to Git
3. SSH into Clouding or use GitHub Actions over SSH
4. `git pull`
5. build image
6. run migrations
7. restart stack
8. verify dashboard
9. trigger one manual pipeline run
10. take a snapshot before major schema changes

Clouding also exposes an API for server lifecycle and infrastructure actions, so later you can automate more infra operations if needed. ([clouding.io][3])

### 16. Rollout order

Use this order so you reduce failure risk.

#### Phase A — Infrastructure

* create Clouding server
* attach firewall profile
* enable backups
* install domain/DNS
* take baseline snapshot

#### Phase B — Application shell

* clone repo
* create `.env`
* bring up Postgres + Streamlit + Nginx
* validate HTTPS access

#### Phase C — Database

* run Alembic migrations
* test DB connectivity from app
* validate schema

#### Phase D — Data migration

* upload SQLite files
* run `bootstrap_from_sqlite.py`
* compare record counts and spot-check dashboards

#### Phase E — Scheduling

* install `garmin-pipeline.timer`
* run one manual pipeline execution
* verify `job_run` and `import_batch`

#### Phase F — Operations

* install backup timer
* verify restore procedure
* take production snapshot

### 17. Acceptance criteria for MVP

The Clouding MVP is complete when:

* the dashboard is reachable over HTTPS
* Postgres is not publicly exposed
* SQLite history is loaded into Postgres
* scheduled runs complete without manual intervention
* Garmin and HRV outputs appear in the dashboard
* nightly SQL backups are being written
* one snapshot exists for rollback
* firewall exposes only SSH, HTTP, HTTPS

### 18. Immediate post-MVP improvements

Once this is stable, the first improvements should be:

* move PostgreSQL to a second private Clouding server using a Private Network (VPC)
* add basic auth or SSO in front of the dashboard if it is private
* add GitHub Actions deployment
* add off-server backup copy
* add monitoring/alerts

Clouding documents Private Network (VPC) support, so splitting app and DB later is a natural next step rather than a redesign. ([help.clouding.io][4])

If you want, I can now rewrite the **Milestones** section for Clouding.io in the same concrete style.

[1]: https://help.clouding.io/hc/en-us/articles/360010598000-Docker-preinstalled-image?utm_source=chatgpt.com "Docker preinstalled image"
[2]: https://help.clouding.io/hc/en-us/articles/13963656119580-How-to-configure-Django-with-Postgres-Nginx-and-Gunicorn-on-Ubuntu-24-04?utm_source=chatgpt.com "How to configure Django with Postgres, Nginx, and ..."
[3]: https://clouding.io/en/features?utm_source=chatgpt.com "The best features for the best Cloud VPS Server"
[4]: https://help.clouding.io/hc/en-us/articles/14352423461788-How-to-add-a-Private-Network-VPC?utm_source=chatgpt.com "How to add a Private Network (VPC)"
