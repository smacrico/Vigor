# Vigor

# On the server, deploy into:

/opt/garmin-stack/
  app/           # git checkout
  postgres/      # postgres data volume or bind mount
  imports/       # Garmin export files
  exports/       # generated HTML reports
  backups/       # pg_dump backups
  nginx/         # nginx config

  # The pipeline runs on demand through 
  docker compose run --rm pipeline-runner ...

  # First boot of the stack
  From /opt/garmin-stack/app/deploy:
  docker compose up -d --build

  # Use Alembic and run migrations as part of every deploy.
  docker compose run --rm dashboard-web alembic upgrade head

  # Bootstrap data from SQLite (one time migration)
  docker compose run --rm \
  -v /opt/garmin-stack/imports:/data/imports \
  dashboard-web \
  python scripts/bootstrap_from_sqlite.py

  # install services
  sudo cp deploy/systemd/garmin-pipeline.service /etc/systemd/system/
sudo cp deploy/systemd/garmin-pipeline.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now garmin-pipeline.timer
# check installed services
systemctl list-timers --all | grep garmin
journalctl -u garmin-pipeline.service -n 100 --no-pager

