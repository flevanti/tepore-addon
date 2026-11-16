# Tepore

A local-first heating engine that runs beside Home Assistant. Every heating decision is made in your house, with no
vendor cloud in the control loop.

## What this version does

This is an early release. It does not control any heating yet. It:

- starts, opens its local database and records a heartbeat every 5 minutes;
- reads Home Assistant's devices (thermostats, TRVs, temperature and humidity sensors, window and presence sensors)
  and follows their state; at `log_level: debug` the add-on log lists them and every change;
- shows a placeholder page ("engine running") in the Home Assistant sidebar.

It writes nothing to any device. Its only writes to Home Assistant are persistent notifications: one when the add-on
stops on a problem it can explain (what is wrong and what to do, at most once an hour for the same problem), and one
when a backup's engine data could not be restored. A fixed problem's notification is cleared at the next healthy start.

For this, the add-on uses Home Assistant's API through the Supervisor (`homeassistant_api`).

## Options

| Option | Default | Meaning |
|---|---|---|
| `log_level` | `info` | Minimum log level: `debug`, `info`, `warn` or `error`. |
| `retention_months` | `12` | How many months of decision records to keep (1 to 120). Older records are deleted. |
| `include_store_in_backups` | `false` | Whether Home Assistant backups include a copy of the engine's database. |

An invalid option stops the add-on at startup; the add-on log says which option is wrong.

## Backups

The live database is never put in a backup: copying it while the engine writes could produce a broken copy.

- **Option off (the default):** backups contain no engine data. Restoring one starts the engine with a fresh,
  empty database.
- **Option on:** just before each backup the engine writes a consistent snapshot of its database, the backup
  includes that snapshot, and the snapshot is deleted again once the backup is done.

Restoring is automatic. When the add-on starts with no database but with a snapshot from a restored backup, it checks
the snapshot and uses it. If the snapshot is damaged, or was written by a newer version of the add-on, the engine
starts with a fresh database instead and keeps the rejected snapshot at `/data/backup/engine.db.rejected`, so nothing
is lost. The add-on stays up either way.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Stopped normally |
| 1 | Unexpected error (please report it, with the add-on log) |
| 2 | Invalid options (see below) |
| 3 | Database from a newer version (see below) |
| 4 | Database problem (see below) |
| 5 | The engine could not serve its page (see below) |
| 6 | The backup snapshot could not be written (Home Assistant reports it in the backup's log) |
| 7 | `engine devices` could not read Home Assistant (developers only) |

## If the add-on stops with exit code 2

The options file could not be used: an option has a value the engine does not accept, or the file is unreadable. The
notification and the add-on log name the option. Correct it in the add-on's Configuration tab and start the add-on. If
the file itself is unreadable, saving the Configuration tab writes it again.

## If the add-on stops with exit code 3

Exit code 3 means the database was written by a newer version of the add-on than the one installed, typically after
going back to an older version. The engine refuses to open it rather than risk damaging it.

To recover, reinstall the newer version (or later): it opens the database as it is.

Going back to an older version is only possible by restoring a backup of that older version taken with
`include_store_in_backups` on. Without such a backup, the older version can only start with a fresh database.

## If the add-on stops with exit code 4

The engine could not open or read its database (the notification names the step and the error). You have two
choices:

- **Restore a Home Assistant backup** that includes Tepore's data (taken with `include_store_in_backups` on).
- **Start fresh, keeping the old database aside.** The add-on's data lives on the Home Assistant host at
  `/mnt/data/supervisor/addons/data/<slug>/store` (the slug is in the add-on page's address, for example
  `abcd1234_tepore`), which needs host access (HA OS debug SSH on port 22222). With the Tepore add-on stopped, move the
  database files aside and start the add-on again:

  ```
  cd /mnt/data/supervisor/addons/data/<slug>/store
  mv engine.db engine.db.bad
  mv engine.db-wal engine.db-wal.bad 2>/dev/null
  mv engine.db-shm engine.db-shm.bad 2>/dev/null
  ```

  The engine then starts with an empty database. The `.bad` files are kept, so nothing is lost.

## If the add-on stops with exit code 5

The engine could not serve its page: another process holds its port, or serving failed. Restart the add-on; if it
happens again, report it with the add-on log.
