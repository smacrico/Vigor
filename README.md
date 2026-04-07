# Vigor

# On the server, deploy into:

/opt/garmin-stack/
  app/           # git checkout
  postgres/      # postgres data volume or bind mount
  imports/       # Garmin export files
  exports/       # generated HTML reports
  backups/       # pg_dump backups
  nginx/         # nginx config