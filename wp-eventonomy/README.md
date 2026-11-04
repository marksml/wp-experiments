# wp-eventonomy

A local WordPress playground for trying out the [Eventonomy](https://eventonomy.org/) events plugin.

This folder contains a Docker Compose setup with two containers:

| Service     | Image               | Purpose                                   | Host access              |
| ----------- | ------------------- | ----------------------------------------- | ------------------------ |
| `wordpress` | `wordpress:7.1.4`   | WordPress 7.1.4 (Apache + PHP)            | http://localhost:8080    |
| `db`        | `mariadb:11.4`      | Database for WordPress                    | `localhost:3306`         |

WordPress core only supports MySQL/MariaDB — PostgreSQL is **not** supported — so MariaDB is used.

Both containers store their data in named Docker volumes (`wordpress_data`, `db_data`), so the site and database survive container restarts and rebuilds. Deleting the volumes resets everything.

## Prerequisites

- Docker Desktop (or Docker Engine + Compose v2) running
- Ports `8080` and `3306` free on the host

## Start

From this folder:

```powershell
docker compose up -d
```

Then open http://localhost:8080 and complete the WordPress installation wizard (site title, admin user, password).

To follow the logs while it starts:

```powershell
docker compose logs -f
```

Wait until MariaDB reports healthy before the WordPress container starts; this is handled automatically by the healthcheck and `depends_on` condition.

## Stop

Stop the containers but keep all data:

```powershell
docker compose stop
```

Resume them later with:

```powershell
docker compose start
```

Or stop and remove the containers (volumes are kept, so data is still safe):

```powershell
docker compose down
```

## Rebuild

Rebuild and recreate the containers, for example after editing `docker-compose.yml` or changing the image tag:

```powershell
docker compose up -d --build --force-recreate
```

If the image tag changed (e.g. a newer WordPress version), also pull the new image first:

```powershell
docker compose pull
docker compose up -d --force-recreate
```

> Note: WordPress core lives on the `wordpress_data` volume. Changing the image tag alone does not overwrite existing files in that volume. To move to a new WordPress version, update through wp-admin (Dashboard → Updates), or reset the volumes and reinstall (see below).

## Reset everything

To delete the containers **and** all site/database data and start fresh:

```powershell
docker compose down -v
docker compose up -d
```

## Accessing the database with DBeaver

Create a new MariaDB/MySQL connection:

| Setting  | Value       |
| -------- | ----------- |
| Host     | `localhost` |
| Port     | `3306`      |
| Database | `wordpress` |
| Username | `wordpress` |
| Password | `wordpress` |

The `root` user is also available with password `root` (server host `%`), if you need full privileges. Credentials are defined in `docker-compose.yml` and are intentionally simple because this is a local-only setup.

## Installing Eventonomy

Eventonomy requires WordPress 6.5+ and PHP 8.1+, both satisfied by this setup.

### From the plugin catalog

1. Open http://localhost:8080/wp-admin
2. Go to **Plugins → Add New Plugin**
3. Search for **Eventonomy**, then **Install** and **Activate**
4. Follow the plugin's setup wizard (it also offers a demo seeder to fill the calendar with sample events)

### From a ZIP file (manual upload)

If the plugin is not in the catalog (e.g. a Pro build), upload the ZIP:

1. Open http://localhost:8080/wp-admin
2. Go to **Plugins → Add New Plugin → Upload Plugin**
3. Choose the ZIP file, click **Install Now**, then **Activate**

WordPress rejects uploads larger than PHP's `upload_max_filesize` / `post_max_size`. This setup raises both limits to `128M` via `uploads.ini`, which is mounted into the container:

```ini
upload_max_filesize = 128M
post_max_size = 128M
memory_limit = 256M
max_execution_time = 300
```

After changing `uploads.ini`, recreate the container so PHP picks it up:

```powershell
docker compose up -d --force-recreate wordpress
```

Verify the new limits under **Tools → Site Health → Info → Media Handling**, or from the command line:

```powershell
docker compose exec wordpress php -i | Select-String "upload_max_filesize|post_max_size"
```

If the ZIP is still too large, raise the values in `uploads.ini` further and recreate the container again.

## Useful commands

```powershell
docker compose ps                 # show container status
docker compose logs -f wordpress  # WordPress/Apache logs
docker compose logs -f db         # MariaDB logs
docker compose exec wordpress bash   # shell inside the WordPress container
docker compose exec db mariadb -uwordpress -pwordpress wordpress   # SQL prompt
```

## Troubleshooting

- **Port already in use**: edit the left side of the `ports:` mappings in `docker-compose.yml` (e.g. `8081:80` or `3307:3306`) and run `docker compose up -d`.
- **WordPress cannot connect to the database**: make sure the `db` service is healthy (`docker compose ps`) and that `WORDPRESS_DB_*` values match the `MARIADB_*` values.
- **Plugin install asks for FTP credentials**: the compose file already sets `FS_METHOD` to `direct`, so this should not happen; if it does, confirm the container was recreated after the setting was added.
