# Task Manager API — Dockerized

A Node.js/Express **Task Manager API** backed by MySQL, containerized with
Docker and Docker Compose.

## Stack

- **app** — Node 20 (alpine), runs the Express API (`server.js`)
- **mysql** — MySQL 8.0, with the `tasks` table auto-created from `db/init.sql`
- Both services sit on a custom bridge network (`app-network`) and reach each
  other by service name (`DB_HOST=mysql`) — no hardcoded IPs or `localhost`
- MySQL data persists in a named volume (`mysql-data`)
- `app` waits for MySQL to be truly ready via a healthcheck (`depends_on:
  condition: service_healthy`), not just started

## Files

```
server.js            # Express app (routes: /, /health, /tasks)
db.js                # MySQL connection pool
package.json
db/init.sql           # creates the "tasks" table
.env                   # working environment values
.env.example            # template for the above
Dockerfile               # builds the Node app image
docker-compose.yml        # app + mysql services
```

## How the Dockerfile works

```dockerfile
FROM node:20-alpine          # small, official Node 20 base image
WORKDIR /app                 # everything below runs from /app in the container

COPY package.json package-lock.json* ./
RUN npm install --production # deps installed BEFORE the rest of the source is
                              # copied, so this layer is cached and only
                              # re-runs when package.json actually changes

COPY . .                     # now copy the rest of the app source

EXPOSE 3000                  # documents the port the app listens on
CMD ["node", "server.js"]    # matches the "start" script in package.json
```

## How docker-compose.yml works

```yaml
services:
  app:
    build: .                       # build the image from the Dockerfile above
    env_file: .env                 # inject PORT / DB_* vars into the container
    ports:
      - "3000:3000"                # host:container port mapping
    depends_on:
      mysql:
        condition: service_healthy # wait for MySQL to be ready, not just started
    networks:
      - app-network

  mysql:
    image: mysql:8.0                       # pinned version, not "latest"
    environment:                            # root password / db / user, matched
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}   # to the app's DB_* values
      MYSQL_DATABASE: ${MYSQL_DATABASE}             # in .env
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql  # auto-creates the
                                                              # "tasks" table
                                                              # on first boot
      - mysql-data:/var/lib/mysql            # named volume — data survives
                                              # container removal
    healthcheck:                             # lets "app" know when MySQL can
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 5s                           # actually accept connections
      timeout: 5s
      retries: 10
    networks:
      - app-network

networks:
  app-network:      # custom bridge network — lets "app" reach MySQL by the
    driver: bridge   # service name "mysql" instead of an IP or localhost

volumes:
  mysql-data:        # declares the named volume used above
```

## Instructions — running it step by step

1. **(Optional) Create your own `.env`** — a working one is already committed, but if starting fresh:
   ```bash
   cp .env.example .env
   ```
2. **Build and start both containers**
   ```bash
   docker compose up --build
   ```
   This builds the `app` image, pulls `mysql:8.0`, starts MySQL first, waits
   for its healthcheck to pass, then starts `app`.
3. **Confirm both containers are healthy**
   ```bash
   docker compose ps
   ```
4. **Test the API** (see [Endpoints](#endpoints) below).
5. **Stop the containers when done**
   ```bash
   docker compose down        # keeps the mysql-data volume
   # or
   docker compose down -v     # also deletes the volume (fresh DB next time)
   ```

## Endpoints

| Method | Route     | Description                              |
|--------|-----------|-------------------------------------------|
| GET    | `/`       | Basic info                                |
| GET    | `/health` | Returns 200 if the app can reach MySQL    |
| GET    | `/tasks`  | Lists all tasks from the DB               |
| POST   | `/tasks`  | Creates a task (`{ "title": "..." }`)     |

```bash
curl http://localhost:3000/health
curl http://localhost:3000/tasks
curl -X POST http://localhost:3000/tasks \
  -H "Content-Type: application/json" \
  -d '{"title": "Finish Docker assignment"}'
curl http://localhost:3000/tasks
```

## Persistence

```bash
# keeps the mysql-data volume — tasks survive the restart
docker compose down
docker compose up

# removes the volume — fresh, empty database
docker compose down -v
docker compose up
```
