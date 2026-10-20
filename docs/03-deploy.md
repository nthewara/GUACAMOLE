# 03 — Deploying the stack

The full Guacamole stack has **three** containers:

1. **`guacd`** — the proxy daemon (our custom FreeRDP 3.x build)
2. **`guacamole`** — the web application (official `guacamole/guacamole` image)
3. **a database** — MySQL/MariaDB or PostgreSQL, holding users + connections

> **Best practice:** put all three on a **user-defined Docker bridge network**
> so they resolve each other by container name via Docker's built-in DNS. Avoid
> the legacy `--link` flag — it is brittle and breaks when you recreate a
> container (a recreated container gets a new internal state, and links bound to
> the old container fail with *"Cannot link to a non running container"*).

## 1. Create the network

```bash
docker network create guac-net
```

## 2. Start the custom guacd

```bash
docker run -d --name guacd --restart always --network guac-net \
  guacd:1.6.1-staging-freerdp3.32
```

It listens on port **4822** (internal to `guac-net`; no host port needed).

## 3. Start the database (example: MySQL)

Use your existing DB if you already have one. For a fresh setup you must
initialise the Guacamole schema first (see the official Guacamole manual —
`initdb.sh` generates the SQL). Attach it to `guac-net`:

```bash
docker network connect guac-net <your-mysql-container>
```

## 4. Start the web application

```bash
docker run -d --name guacamole --restart always --network guac-net \
  -p 8080:8080 \
  -e GUACD_HOSTNAME=guacd \
  -e MYSQL_HOSTNAME=<your-mysql-container-name> \
  -e MYSQL_DATABASE=guacamole_db \
  -e MYSQL_USERNAME=guacamole_user \
  -e MYSQL_PASSWORD=<your-db-password> \
  guacamole/guacamole:latest
```

Then browse to `http://<host>:8080/guacamole/`.

> **Env var note (Guacamole 1.6.0):** the DB **username** var was renamed from
> `MYSQL_USER` to **`MYSQL_USERNAME`** (because `MYSQL_USER` collided with the
> MySQL image's own variable). The old name still works but prints a deprecation
> warning. `MYSQL_PASSWORD`, `MYSQL_DATABASE`, `MYSQL_HOSTNAME` are unchanged.

## Docker Compose

A sanitized reference compose file is provided at
[`../docker-compose.example.yml`](../docker-compose.example.yml). Fill in your
own passwords/secrets (use a `.env` file or Docker secrets — never commit them).

## Upgrading / swapping guacd safely

Because the custom guacd is a **local image** it does not auto-update. To swap in
a rebuilt version with zero-fuss rollback:

```bash
# keep the old one as a stopped fallback
docker stop guacd && docker rename guacd guacd-backup

# start the new build
docker run -d --name guacd --restart always --network guac-net \
  guacd:1.6.1-staging-freerdp3.32

# once confirmed working, remove the fallback
docker rm guacd-backup
```

> Because everything is on `guac-net` with name-based DNS, the web app finds the
> new `guacd` automatically — no need to touch the web container. (If you ever
> used `--link` instead, you'd have to recreate the web app too. Don't use
> `--link`.)

## Common deploy pitfall: stale web container

If the `guacamole` web container was created long ago from an old image tag, a
`docker restart` will **not** pick up a newer pulled image — Docker freezes the
old image's config (including its `JAVA_HOME`) into the container. Symptom:

```
The JAVA_HOME environment variable is not defined correctly
JAVA_HOME=/usr/local/openjdk-8
```

**Fix:** actually **recreate** the container (`docker rm -f` + `docker run`, or
in Portainer use *Recreate* with *re-pull image*) so it boots from the current
image (1.6.0 uses JDK 21 at `/opt/java/openjdk`). "Always pull image" only
applies on (re)deploy, not to an already-running container.
