# Orthanc + PostgreSQL (Prod) — Quick Start

1) Create `.env` from `.env.example`. Set strong secrets.
2) Set DNS for `ORTHANC_HOST` and `TRAEFIK_DASHBOARD_HOST` to this server.
3) `mkdir -p orthanc` and save `orthanc/orthanc.json`.
4) `docker compose pull && docker compose up -d`.
5) First login: https://$ORTHANC_HOST (user: admin). Change password.
6) Open DICOM (4242/TCP) only to trusted networks or via VPN. RadiAnt AE → send to AET=ORTHANC, HOST=<server>, PORT=4242.

# Backups
- PostgreSQL (metadata): nightly logical dumps in `pg_backups` volume. Copy offsite or sync to S3.
- Orthanc storage directory: nightly restic backup to `${RESTIC_REPOSITORY}` with retention. For S3/MinIO set AWS creds; for local repo bind-mount a host path.

# Restore
- DB: `docker run --rm -v pg_backups:/backups -e RESTORE_FILE=<dump> postgres:16 bash -lc "pg_restore -h postgres -U $POSTGRES_USER -d $POSTGRES_DB /backups/$RESTORE_FILE"`
- Storage: `restic restore latest --target /restore` then replace `orthanc_storage` content while Orthanc is stopped.

# Scaling & Hardening
- Put stack behind site-to-site VPN or ZeroTier/Tailscale.
- Optionally enable Orthanc PostgreSQL storage (`EnableStorage=true`) if you prefer DB-only backups; then switch backups to PITR (pgBackRest) instead of logical dumps.
- Consider WORM/immutable retention on object storage.
- Monitor cadvisor on :8080 or ship logs to ELK.

# Migration
- Bring stack down; snapshot/copy volumes: `pg_data`, `pg_backups`, `orthanc_storage`, `traefik_letsencrypt`.
- Move `.env`, `docker-compose.yml`, `orthanc/` to new host.
- Bring up; adjust DNS. Done.
