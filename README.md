---
title: "Grimmory on FreeBSD: Self-Hosted Book Library with Kobo & KOReader Sync"
description: "Deploy Grimmory on FreeBSD natively using Podman. Self-hosted book management with smart shelves, metadata lookup, built-in reader, and Kobo/KOReader sync — successor to BookLore."
---

# :material-bookshelf: Grimmory

[![Build Status](https://img.shields.io/github/actions/workflow/status/daemonless/grimmory/build.yaml?style=flat-square&label=Build&color=green)](https://github.com/daemonless/grimmory/actions)
[![Last Commit](https://img.shields.io/github/last-commit/daemonless/grimmory?style=flat-square&label=Last+Commit&color=blue)](https://github.com/daemonless/grimmory/commits)

Self-hosted book management — successor to BookLore. Organize your library with smart shelves, built-in reader, metadata lookup, and sync to Kobo and KOReader devices.

| | |
|---|---|
| **Registry** | `ghcr.io/daemonless/grimmory` |
| **Source** | [https://github.com/grimmory-tools/grimmory](https://github.com/grimmory-tools/grimmory) |
| **Website** | [https://grimmory.org/](https://grimmory.org/) |

## Features

- **Smart Shelves** — Rule-based filtering, tagging, and full-text search
- **Built-in Reader** — In-browser reading for PDFs, EPUBs, and comics with annotations and progress tracking
- **Metadata Lookup** — Book info from Google Books, Open Library, and Amazon (fully editable)
- **Device Sync** — Kobo compatibility, OPDS support, and KOReader synchronization
- **BookDrop** — Drop files into a watched folder; Grimmory detects, enriches metadata, and queues for review
- **Multi-User** — Individual shelves and preferences with local or OIDC authentication
- **One-Click Sharing** — Send books to Kindle, email, or other users

**Supported formats:** EPUB, MOBI, AZW, AZW3, FB2, PDF, CBZ, CBR, CB7, M4B, M4A, MP3, OPUS

## Version Tags

| Tag | Description | Best For |
| :--- | :--- | :--- |
| `latest` | **Upstream Binary**. Built from official release. | Most users. Matches Linux Docker behavior. |

## Prerequisites

Before deploying, ensure your host environment is ready. See the [Quick Start Guide](https://daemonless.io/guides/quick-start) for host setup instructions.

## Deploy

=== ":material-tune: Podman Compose"

    Save as `compose.yaml` and run `podman-compose up -d`:

    ```yaml { data-zip-bundle="grimmory-podman" data-zip-filename="compose.yaml" }
    name: grimmory

    services:
      grimmory:
        image: ghcr.io/daemonless/grimmory:latest
        container_name: grimmory
        network_mode: host
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Etc/UTC
          - DATABASE_URL=jdbc:mariadb://127.0.0.1:3306/grimmory
          - DATABASE_USERNAME=grimmory
          - DATABASE_PASSWORD=changeme
          - SWAGGER_ENABLED=false
          - FORCE_DISABLE_OIDC=false
        volumes:
          - grimmory-data:/app/data
          - /path/to/books:/books
          - grimmory-bookdrop:/bookdrop
        depends_on:
          - mariadb
        restart: unless-stopped

      mariadb:
        image: ghcr.io/daemonless/mariadb:latest
        container_name: grimmory-mariadb
        network_mode: host
        environment:
          - PUID=1000
          - PGID=1000
          - TZ=Etc/UTC
          - MYSQL_ROOT_PASSWORD=changeme
          - MYSQL_DATABASE=grimmory
          - MYSQL_USER=grimmory
          - MYSQL_PASSWORD=changeme
        volumes:
          - grimmory-mariadb:/config
        restart: unless-stopped

    volumes:
      grimmory-data:
      grimmory-bookdrop:
      grimmory-mariadb:
    ```

    Access Grimmory at **http://your-host:6060**

=== ":material-console: Podman CLI"

    ```bash
    # Start MariaDB first
    podman run -d --name grimmory-mariadb \
      --network host \
      -e PUID=1000 -e PGID=1000 -e TZ=Etc/UTC \
      -e MYSQL_ROOT_PASSWORD=changeme \
      -e MYSQL_DATABASE=grimmory \
      -e MYSQL_USER=grimmory \
      -e MYSQL_PASSWORD=changeme \
      -v /path/to/containers/grimmory/mariadb:/config \
      ghcr.io/daemonless/mariadb:latest

    # Then start Grimmory
    podman run -d --name grimmory \
      --network host \
      -e PUID=1000 -e PGID=1000 -e TZ=Etc/UTC \
      -e DATABASE_URL=jdbc:mariadb://127.0.0.1:3306/grimmory \
      -e DATABASE_USERNAME=grimmory \
      -e DATABASE_PASSWORD=changeme \
      -v /path/to/containers/grimmory/data:/app/data \
      -v /path/to/books:/books \
      -v /path/to/containers/grimmory/bookdrop:/bookdrop \
      ghcr.io/daemonless/grimmory:latest
    ```

## Parameters

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PUID` | `1000` | User ID |
| `PGID` | `1000` | Group ID |
| `TZ` | `Etc/UTC` | Timezone |
| `DATABASE_URL` | `jdbc:mariadb://127.0.0.1:3306/grimmory` | MariaDB JDBC connection URL |
| `DATABASE_USERNAME` | `grimmory` | Database username |
| `DATABASE_PASSWORD` | `changeme` | Database password |
| `SWAGGER_ENABLED` | `false` | Enable Swagger UI |
| `FORCE_DISABLE_OIDC` | `false` | Force-disable OIDC authentication |
| `DISK_TYPE` | — | Set to `NETWORK` for NFS/SMB mounts (disables destructive file ops) |
| `JAVA_TOOL_OPTIONS` | — | Additional JVM options (e.g., `-Xmx512m` to cap memory) |

### Volumes

| Path | Description |
|------|-------------|
| `/app/data` | Application data and configuration |
| `/books` | Book library directory |
| `/bookdrop` | Drop folder for automatic imports |

### Ports

| Port | Description |
|------|-------------|
| `6060` | Web interface |

## Networking

The default compose uses `network_mode: host` — both containers communicate via `127.0.0.1`. Only port 6060 needs to be exposed externally.

For isolated networking (multiple stacks, no port conflicts), use bridge mode with the [dnsname CNI plugin](https://github.com/containers/dnsname):

```yaml
services:
  grimmory:
    # remove network_mode: host, add:
    ports:
      - "6060:6060"
    environment:
      DATABASE_URL: "jdbc:mariadb://mariadb:3306/grimmory"

  mariadb:
    # remove network_mode: host
    # container name becomes the DNS hostname
```

## Migration from BookLore

Grimmory is the direct successor to BookLore and auto-migrates existing data on first start.

1. Stop the BookLore container
2. Copy `/containers/booklore/` to `/containers/grimmory/`
3. Deploy Grimmory pointing at the same MariaDB data
4. Grimmory will migrate the schema on startup

The MariaDB data format is compatible between BookLore and Grimmory.

## BookDrop

BookDrop watches the `/bookdrop` volume for new files. Drop any supported file in and Grimmory will:

1. Detect the new file
2. Extract and look up metadata
3. Queue it for your review before adding to the library

**NFS/SMB tip:** If `/bookdrop` is on a network share, set `DISK_TYPE=NETWORK` to prevent destructive file operations on shared mounts.

**Architectures:** amd64
**User:** `bsd` (UID/GID via PUID/PGID, defaults to 1000:1000)
**Base:** FreeBSD 15.0

---

Need help? Join our [Discord](https://discord.gg/Kb9tkhecZT) community.
