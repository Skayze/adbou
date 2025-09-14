# Quick Start
1) Copy `.env.example` → `.env` and set:
   - `ONEDRIVE_ROOT` to a folder inside your OneDrive that will contain:
     `orthanc-storage/` and `pg-backups/` (create both).
   - DB creds and `ORTHANC_ADMIN_PASSWORD`.
2) Save `orthanc/orthanc.json` (this file) and `docker-compose.yml` side-by-side.
3) `docker compose pull && docker compose up -d`.
4) Access Orthanc UI: http://localhost:8042 (user: admin; password from `.env`).
5) RadiAnt: send to AET=ORTHANC, HOST=127.0.0.1, PORT=4242.

# Backups
- PostgreSQL: nightly `pg_dump` (custom format) into `${ONEDRIVE_ROOT}/pg-backups`.
- DICOM files: written directly into `${ONEDRIVE_ROOT}/orthanc-storage`, so OneDrive syncs them.

# Restore / Migration to New Host
- Install Docker.
- Install/sign in to OneDrive and let it sync the same `${ONEDRIVE_ROOT}` tree.
- Copy this repo (`docker-compose.yml`, `.env`, `orthanc/`) to the new host.
- `docker compose up -d` (ensure `.env` paths point to the synced location).
- Restore DB if needed: `pg_restore -h postgres -U $POSTGRES_USER -d $POSTGRES_DB <dumpfile>`.

# Important Caveats with OneDrive (read before production)
1) **Do NOT place the live Postgres data directory in OneDrive** — corruption risk.
2) **IndexDirectory must NOT be in OneDrive.** It’s a LevelDB; syncing/open-file conflicts can corrupt it.
3) **Performance:** OneDrive may slow down writes for many small files (DICOM chunks). Local SSD is faster.
4) **Consistency:** OneDrive is a file synchronization tool, not a true backup. There is no point-in-time snapshot of the *combined* DB + storage. If a failure happens mid-sync, file and DB states can diverge.
5) **Open files/locking:** Long transfers may create partial uploads or temp files; avoid killing power mid-send.
6) **Versioning limits:** OneDrive retains limited versions and may not keep every state for large, hot trees.

# Safer patterns while keeping OneDrive
- Keep Orthanc storage on local disk and run a nightly **snapshot** into an archive inside OneDrive, e.g.:
  - `robocopy` (Windows) or `rsync` (Linux) to a date-stamped folder, or
  - use `restic` with repo hosted in the OneDrive folder (gives dedup + integrity checks + retention), or
  - compress `orthanc-storage/` to a `.zip/.7z` with a checksum next to it.
- Always pair the storage snapshot with the matching Postgres dump from the same night.
- For migrations/restores, stop containers, ensure OneDrive finished syncing, then proceed.

# Switching later to Traefik/HTTPS or DICOMweb
- Add Traefik service and labels; put Orthanc behind TLS; optionally enable DICOMweb plugin and auth.
	