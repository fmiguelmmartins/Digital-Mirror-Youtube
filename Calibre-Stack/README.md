# Calibre + Calibre-Web Docker Compose Stack

A simple two-container setup that runs:

- **Calibre** — the main desktop-style Calibre interface used to manage your library.
- **Calibre-Web** — a lightweight web interface used to browse, search, download, and organise the same Calibre library.

Both applications share the same Calibre library and `metadata.db`, while keeping their own application configuration separate.

## Requirements

You need:

- Docker
- Docker Compose
- An x86-64 or ARM64 Linux host
- A 64-bit Raspberry Pi OS if running on a Raspberry Pi

## Start the stack

Place `docker-compose.yml` in a folder of your choice, open a terminal in that folder, and run:

```bash
docker compose up -d
```

Docker will download the images and start both applications.

Check that they are running with:

```bash
docker compose ps
```

## Default folder layout

With the supplied Compose file, Docker stores persistent data in these host folders:

```text
.
├── docker-compose.yml
├── calibre-stack/
│   ├── calibre-main/
│   │   └── config/
│   ├── calibre-web/
│   │   └── config/
│   └── library/
└── calibre-main/
    └── ingest_books/
```

The important folders are:

| Host folder | Purpose |
|---|---|
| `./calibre-stack/calibre-main/config` | Calibre application settings |
| `./calibre-stack/calibre-web/config` | Calibre-Web settings, users, shelves, `app.db`, etc. |
| `./calibre-stack/library` | The actual Calibre library: `metadata.db`, author folders, books and covers |
| `./calibre-main/ingest_books` | Convenient folder exposed inside Calibre for importing books |

If you prefer to keep everything somewhere else, replace the host-side paths in `docker-compose.yml` with your own absolute paths.

For example:

```yaml
- /mnt/ssd/appdata/calibre/calibre-main/config:/config
- /mnt/ssd/appdata/calibre/library:/config/Calibre Library
```

Only change the path on the **left** side of `:` unless you know you need a different container path.

## First Calibre setup

Open:

```text
https://YOUR-SERVER-IP:8181
```

Your browser may warn about the local self-signed certificate. Accept the warning for your trusted local network.

The first time Calibre starts, the Welcome Wizard asks where the library should be stored.

Use:

```text
/config/Calibre Library
```

The Compose file maps that container path to:

```text
./calibre-stack/library
```

Calibre will create `metadata.db` there automatically.

### Existing Calibre library

If you already have a Calibre library, stop the stack first:

```bash
docker compose down
```

Copy the **entire contents** of your existing Calibre library into:

```text
./calibre-stack/library
```

That directory should look roughly like:

```text
library/
├── metadata.db
├── Author Name/
│   └── Book Title/
│       ├── Book Title.epub
│       ├── cover.jpg
│       └── metadata.opf
└── ...
```

Then start the stack again:

```bash
docker compose up -d
```

Do not copy only `metadata.db`; keep it together with the Calibre-managed author/book directory structure.

## Calibre-Web setup

Open:

```text
http://YOUR-SERVER-IP:8083
```

The default Calibre-Web credentials are:

```text
Username: admin
Password: admin123
```

Change the password after logging in.

When Calibre-Web asks for **Location of Calibre Database**, enter:

```text
/books
```

The Compose file maps:

```text
./calibre-stack/library
```

to:

```text
/books
```

inside the Calibre-Web container.

That means both applications use the same library:

```text
Host
./calibre-stack/library
        │
        ├── Calibre     → /config/Calibre Library
        └── Calibre-Web → /books
```

Both therefore see the same `metadata.db` and the same books.

## Import / ingest folder

The Compose file also maps:

```text
./calibre-main/ingest_books
```

to:

```text
/digitalmirror/ingest_books
```

inside the Calibre container.

You can place books in the host folder:

```text
./calibre-main/ingest_books
```

and browse to this location from Calibre when importing:

```text
/digitalmirror/ingest_books
```

This folder is simply a convenient import/drop location. It is not the Calibre library itself.

## PUID and PGID

The supplied Compose file uses:

```yaml
PUID: "1000"
PGID: "1000"
```

These should normally match the Linux user that owns the host folders.

Check your current user IDs with:

```bash
id
```

Example:

```text
uid=1000(user) gid=1000(user)
```

In that case, leave the Compose file as:

```yaml
PUID: "1000"
PGID: "1000"
```

If your IDs are different, update both services to use the correct values.

Calibre and Calibre-Web should use the **same PUID and PGID** because they both access the shared library.

## Time zone

The supplied configuration uses:

```yaml
TZ: Europe/London
```

Change this if required.

For example:

```yaml
TZ: Europe/Lisbon
```

## Ports

The default ports are:

| Application | URL |
|---|---|
| Calibre | `https://SERVER-IP:8181` |
| Calibre-Web | `http://SERVER-IP:8083` |

If either host port is already in use, change only the number on the **left** side.

Example:

```yaml
ports:
  - "8282:8181"
```

Calibre would then be available at:

```text
https://SERVER-IP:8282
```

The container still listens on port `8181`.

## NAS and network storage

Local NAS datasets can be used directly as Docker bind mounts.

For example:

```yaml
volumes:
  - /mnt/ssd/appdata/calibre-main/config:/config
  - /mnt/tank/books/calibre-library:/config/Calibre Library
```

Calibre-Web would mount the same library:

```yaml
volumes:
  - /mnt/ssd/appdata/calibre-web/config:/config
  - /mnt/tank/books/calibre-library:/books
```

There is no special `network-share=true` option required by this LinuxServer Calibre-Web setup for ordinary local NAS datasets.

If your library itself is mounted from NFS, SMB/CIFS, or another remote filesystem, make sure Docker can access that mount and that the configured PUID/PGID has permission to read and write it.

## Stop the stack

Run:

```bash
docker compose down
```

This removes the running containers and Compose network but **does not delete your bind-mounted data folders**.

Start it again with:

```bash
docker compose up -d
```

## Update the containers

Pull the newest images:

```bash
docker compose pull
```

Then recreate the containers:

```bash
docker compose up -d
```

Your library and configuration remain in the host folders.

Optionally remove unused old images:

```bash
docker image prune
```

## View logs

Calibre:

```bash
docker compose logs -f calibre
```

Calibre-Web:

```bash
docker compose logs -f calibre-web
```

All services:

```bash
docker compose logs -f
```

Press `Ctrl+C` to stop following the logs.

## Backups

Back up these folders:

```text
./calibre-stack/calibre-main/config
./calibre-stack/calibre-web/config
./calibre-stack/library
```

The most important files are:

```text
Calibre library:
metadata.db + all author/book folders

Calibre-Web:
app.db + the rest of its config directory
```

Backing up the entire directories is simpler and safer than selecting individual files.

The `ingest_books` folder normally does not need to be backed up if it is only being used as a temporary import folder.

## Complete workflow

The final setup works like this:

```text
                    ┌────────────────────┐
                    │      Calibre       │
                    │                    │
                    │ Library management │
                    │ Metadata editing   │
                    │ Book importing     │
                    └─────────┬──────────┘
                              │
                              │ /config/Calibre Library
                              ▼
                    ┌────────────────────┐
                    │   Shared Library   │
                    │                    │
                    │ metadata.db        │
                    │ books              │
                    │ covers             │
                    └─────────┬──────────┘
                              │
                              │ /books
                              ▼
                    ┌────────────────────┐
                    │    Calibre-Web     │
                    │                    │
                    │ Browser UI         │
                    │ Shelves            │
                    │ Users              │
                    │ Downloads          │
                    └────────────────────┘
```

## Quick start

For a brand-new installation:

```bash
docker compose up -d
```

Then:

1. Open `https://SERVER-IP:8181`.
2. Set the Calibre library to `/config/Calibre Library`.
3. Open `http://SERVER-IP:8083`.
4. Log in with `admin / admin123`.
5. Set **Location of Calibre Database** to `/books`.
6. Change the Calibre-Web admin password.
7. Start adding books.

That is all that is required for the basic stack.
