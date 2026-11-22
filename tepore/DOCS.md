# Tepore

A local-first heating engine that runs beside Home Assistant. Every heating decision is made in your house, with no
vendor cloud in the control loop.

## What this version does

This is an early release. It does not control any heating yet. It:

- starts, opens its local database and records a heartbeat every 5 minutes;
- reads Home Assistant's devices (thermostats, TRVs, temperature and humidity sensors, window and presence sensors)
  and follows their state; at `log_level: debug` the add-on log lists them and every change;
- keeps the links to the Home Assistant entities of bound devices up to date after every full read;
- shows a first page in the Home Assistant sidebar: the engine's status (version, database schema, Home Assistant
  connection) and the home's rooms and devices. In the add-on, rooms cannot be created yet: an empty home is expected.
  To try rooms and devices now, run the engine in Docker (below).

It writes nothing to any device. Its only writes to Home Assistant are persistent notifications: one when the engine
enters safe mode on a problem it can explain (what is wrong and what you can do, at most once an hour for the same
problem). A fixed problem's notification is cleared at the next healthy start.

For this, the add-on uses Home Assistant's API through the Supervisor (`homeassistant_api`).

## Options

| Option | Default | Meaning |
|---|---|---|
| `log_level` | `info` | Minimum log level: `debug`, `info`, `warn` or `error`. |
| `retention_months` | `12` | How many months of decision records to keep (1 to 120). Older records are deleted. |
| `include_store_in_backups` | `false` | Whether Home Assistant backups include a copy of the engine's database. |

An invalid option puts the engine in safe mode at startup; the panel and the add-on log say which option is wrong.

## Backups

The live database is never put in a backup: copying it while the engine writes could produce a broken copy.

- **Option off (the default):** backups contain no engine data. Restoring one starts the engine with a fresh,
  empty database.
- **Option on:** just before each backup the engine writes a consistent snapshot of its database, the backup
  includes that snapshot, and the snapshot is deleted again once the backup is done.

Restoring is automatic. When the add-on starts with no database but with a snapshot from a restored backup, it checks
the snapshot and uses it. If the snapshot is damaged, was written by a newer version of the add-on, or cannot be put in
place, the engine enters safe mode instead: the panel explains it, the snapshot stays where it is, and you choose
between starting fresh (the snapshot is kept at `/data/backup/engine.db.rejected`) and restoring another backup.

## Safe mode

When the engine meets a problem it can explain at startup, it does not stop. It stays up in safe mode: it makes no
decisions, writes nothing to any device, and the Tepore panel shows what is wrong and what you can do. The add-on log
repeats it every 5 minutes, and Home Assistant shows a notification.

| Problem | What you can do |
|---|---|
| An option is invalid, or the options file is unreadable | Correct it in the add-on's Configuration tab (saving rewrites the file) and restart the add-on. |
| The database was written by a newer version | Install that version or a later one: it opens the database as it is. |
| The database cannot be opened or read | **Start fresh** from the panel: the database files are moved aside (renamed with `.bad.` and a time), never deleted, and the engine starts with an empty database. Or restore a Home Assistant backup that includes Tepore's data (taken with `include_store_in_backups` on). |
| A restored backup's data was rejected or could not be put in place | **Start fresh** from the panel (the backup's file is kept), or restore another backup. |
| The check for a restored backup failed | Fix the cause named in the panel (for example a permission) and restart the add-on. |

A version cannot be downgraded once it has updated the database. Going back to an older version is only possible by
restoring a backup of that older version taken with `include_store_in_backups` on. Without such a backup, the older
version can only start with a fresh database.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Stopped normally |
| 1 | Unexpected error (please report it, with the add-on log) |
| 2 | Bad arguments, or `engine devices` with unusable options |
| 5 | The engine could not serve its page (see below) |
| 6 | The backup snapshot could not be written (Home Assistant reports it in the backup's log) |
| 7 | `engine devices` could not read Home Assistant (developers only) |
| 8 | `engine call` could not connect or sign in, or the engine refused the request (Docker) |
| 11 | `engine token reset` could not remove the token file (Docker) |

## If the add-on stops with exit code 5

The engine could not serve its page: another process holds its port, or serving failed. Restart the add-on; if it
happens again, report it with the add-on log.

## Running in Docker without Home Assistant OS

The same image runs on its own, next to any Home Assistant you can reach. This is how to try rooms and devices in this
version. The Docker engine has its own database, separate from any add-on's.

1. Pick the image for your machine: `ghcr.io/flevanti/tepore-amd64` or `ghcr.io/flevanti/tepore-aarch64` (Apple
   silicon, Raspberry Pi), tagged with the version, for example `0.4.0`.
2. Create a writable volume for `/data` and put an `options.json` in it, readable only by you (mode `0600`):

   ```json
   {"ha_url": "http://192.168.1.10:8123", "ha_token": "<long-lived access token>"}
   ```

   The token comes from Home Assistant: your profile → Security → Long-lived access tokens, for an administrator user.
   `ha_url` must be reachable from inside the container: use an IP address when `.local` names do not resolve there.
   The add-on's options (`log_level`, `retention_months`, `include_store_in_backups`) work here too. Do not set
   `SUPERVISOR_TOKEN`: with it the engine behaves as the add-on and refuses every browser.
3. Start it, publishing port 8099:

   ```
   docker run -d --name tepore -v tepore-data:/data -p 8099:8099 ghcr.io/flevanti/tepore-aarch64:0.4.0
   ```

4. Read the one-time setup code from `docker logs tepore` (the line "setup code for the first sign-in"). Open
   `http://localhost:8099`, paste the code, and keep the token the page shows: `engine call` needs it.
5. Configure the home with `engine call`, inside the container. The first time, the setup code can stand in for the
   token, and the token is printed:

   ```
   docker exec -e TEPORE_API_TOKEN=<setup code> tepore engine call engine.status
   docker exec -e TEPORE_API_TOKEN=<token> tepore engine call circuits.create '{"name":"Main","type":"radiator"}'
   ```

   Keep the token out of shell history you share.
6. Lost the token? `docker exec tepore engine token reset`, then `docker restart tepore`, and a new setup code is in
   the log.

The engine serves plain HTTP. Before it leaves your machine, put TLS in front of it with a reverse proxy, which must pass
the original `Host` header (for nginx, `proxy_set_header Host $host;`).
