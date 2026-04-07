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

  