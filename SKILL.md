# Skill: Deploy an app on Copas

Use this when a user asks to deploy an app on **Copas** (which runs on the
**orcinus** cluster engine). The Copas client CLI deploys a single app straight
from **source** with **Railpack** — no Dockerfile, no manifest, no Docker, no
`kubectl`.

> **Always prefer Railpack.** `copas up` auto-detects the stack (Node, Go,
> Python, Ruby, PHP, …) and builds it on the server. Only fall back to a
> Dockerfile when the app genuinely can't be built by Railpack.

## Default (and recommended) path: `copas up`

1. Log in and pick/create a project:

   ```bash
   copas login --server https://copas.example.com
   copas project                       # list projects (alias: projects)
   copas project create <p>            # create one if needed
   ```

2. Inspect the app: language/stack, the **port** it listens on, the **env** it
   needs, and whether it needs a database, cache, or persistent volume.

3. From the app directory, deploy with Railpack (the default):

   ```bash
   copas up --project <p> --name <app>
   ```

   That minimal form already: builds with **Railpack**, exposes the app at
   **`<name>.<appsDomain>` with TLS**, and defaults the **port to 3000** (also
   set as `$PORT`). Add flags as needed:

   ```bash
   copas up --project <p> --name <app> --build railpack \
     --path . --port <port> --env-file .env --env KEY=VALUE --secret KEY \
     --mount data:/var/lib/app --volume-size 1Gi --domain <host> --tls
   ```

   It packs the directory (honors `.dockerignore`, else `.gitignore`; always
   excludes `.git`), uploads it, and **streams the server-side build + deploy
   log**. Follow later with `copas deployment status <id>`.

## Build type

- **`--build railpack`** (default) — no Dockerfile needed; Railpack auto-detects
  the stack. **This is the recommended path — use it unless it can't work.**
- **`--build dockerfile`** (fallback only) — uses the repo's Dockerfile
  (`--dockerfile <path>`, `--context-dir <subdir>` for a subfolder). Reach for
  this only when Railpack can't build the app (unusual stack, custom system
  deps, or a hand-tuned image).

## Notes that matter

- **Monorepos** with several apps: deploy each app separately with
  `--context-dir apps/<name>` — do **not** `up` the repo root (Railpack finds no
  single start command and the container will crash-loop).
- **Databases**: create managed databases (Postgres/MySQL/MariaDB/MongoDB/Redis)
  from the **web UI**, then pass their connection string to the app via `--env`
  / `--env-file`. Deploy the database **before** an app that runs migrations on
  startup so the DB is reachable at deploy time.

  ```
  DATABASE_URL=postgresql://<user>:<pass>@<db-name>:5432/app
  REDIS_URL=redis://<redis-name>:6379
  ```

- Mark sensitive env keys with `--secret KEY` so they render as a Kubernetes
  Secret instead of plain env. Never hard-code long-lived credentials.
- `copas info` prints the server's apps domain, registry host, and public URL so
  you know where apps land and whether in-cluster builds are available.

## Managing deployments

```bash
copas deployment list --project <p>       # deploy history
copas deployment status <deployment-id>   # stream a deployment's log
copas deployment redeploy <deployment-id> # roll back to a past snapshot
```

## Troubleshoot (watch the streamed deploy log)

- **`CrashLoopBackOff`, exits immediately (exit 0)** → Railpack found no start
  command (often a monorepo root with no root `start` script). Point `up` at the
  actual app (`--context-dir apps/<name>`) or set `--command`.
- **`CrashLoopBackOff`** (other) → app started but exited: bad config, DB
  unreachable, failed migration. Check logs; confirm `DATABASE_URL`/host.
- **`ImagePullBackOff` / `ErrImagePull`** → the built image isn't pullable (rare
  with `up`; usually a registry/build config issue on the server).
- **Pod stuck `Pending`** → no schedulable node (resource/disk pressure) or a PVC
  that can't bind.
- **TLS not `Ready`** → the host must resolve to the cluster and :80 be reachable
  for the ACME HTTP-01 challenge.

## Deploy checklist

- [ ] Use `copas up` with **Railpack** (the default); only use `--build dockerfile` if Railpack can't build it.
- [ ] The app's listen port matches `--port`; it binds `$PORT` / `0.0.0.0`.
- [ ] Managed databases created **first**; connection strings passed as env (secrets marked with `--secret`).
- [ ] Public apps get a host + TLS (auto `<name>.<appsDomain>`, or explicit `--domain --tls`).
- [ ] Monorepo: deploy each app with `--context-dir`, not the repo root.
- [ ] The DNS host resolves to the cluster (for TLS issuance).
