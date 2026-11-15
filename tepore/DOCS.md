# Tepore

A local-first heating engine that runs beside Home Assistant. Every heating decision is made in your house, with no
vendor cloud in the control loop.

## What this version does

This is an early foundation release. It does not control any heating yet. It:

- starts, opens its local database and records a heartbeat every 5 minutes;
- shows a placeholder page ("engine running") in the Home Assistant sidebar.

It has no access to Home Assistant's API and does not talk to any device.

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

## If the add-on stops with exit code 3

Exit code 3 means the database was written by a newer version of the add-on than the one installed, typically after
going back to an older version. The engine refuses to open it rather than risk damaging it.

To recover, reinstall the newer version (or later): it opens the database as it is.

Going back to an older version is only possible by restoring a backup of that older version taken with
`include_store_in_backups` on. Without such a backup, the older version can only start with a fresh database.
