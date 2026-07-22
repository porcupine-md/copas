---
name: copas
description: Use when a user wants to deploy, update, inspect, recover, or plan the deployment of a repository on Copas. Copas runs apps on the orcinus cluster engine.
---

# Skill: Operate Copas through the CLI

## The operating rule

For a source repository, **`copas up` is the mandatory deployment path**.

It is not merely an upload command. It resolves or creates the target project, creates or updates the named application, packages source, uploads it, builds on the server, deploys it, and follows the deployment by default:

```text
project → application → source upload → server-side build → deploy → deployment log
```

After reviewing the repository, when the agent is unsure how the CLI is configured or where an application will be exposed, run `copas info`. It reports the active server, apps domain, registry, and public URL; use it to resolve uncertainty before planning flags or asking the human about routine CLI configuration.

Do not substitute Docker, Kubernetes manifests, `kubectl`, or the Web UI for a normal source deployment. `copas up` uses Railpack by default: a server-side builder that detects the application stack and builds source without a Dockerfile. Do not ask the human to choose it. Use a Dockerfile only when review gives a concrete reason the default builder cannot build the application.

## Start here: repository-to-deployment flow

When asked to deploy a repository, do this in order. Do not begin by asking the user for a long configuration questionnaire.

### 1. Review the repository first and form a plan

Repository review is the first action for a deployment request. Before running `copas info`, `copas login`, `copas db`, or `copas up`—and before asking the human for routine deployment choices—inspect the local repository to determine how it builds and deploys. If the repository is unavailable locally, ask for its location or access before planning a deployment.

Read only the evidence needed to determine how to deploy:

- runtime/build markers: `package.json`, lockfiles, `go.mod`, `pyproject.toml`, `requirements.txt`, `Gemfile`, `composer.json`, `Cargo.toml`;
- start/build scripts, entrypoint, exposed port, bind host, and any declared health endpoint (`/health`, `/healthz`, `/ready`, or equivalent);
- `Dockerfile` and Compose files as signals, not as the default deployment path;
- monorepo markers (workspace configuration, several independently deployable apps, or a user statement) and the actual app subdirectory when evidence exists;
- `.env.example`, config templates, migration directories, ORM/framework configuration, and connection-string variable names;
- persistent files, uploads, queues/workers, caches, scheduled jobs, and other service dependencies.

Never print, upload, or request secret values during this inspection. A variable name in an example/config file is useful; a credential is not.

Classify the repository briefly:

- **Simple:** one stateless web app, one runtime, no external state detected.
- **Structured:** monorepo, custom start command, non-default port, or required configuration.
- **Operationally complex:** database migrations, cache/queue, worker process, persistent uploads, several services, or a Compose topology.

This is a structural review, not a capacity guarantee. Do not claim a repository is safe for a certain traffic level without measurements, resource limits, and runtime observability.

Treat the repository as a monorepo only when the user says it is one or review finds evidence. Otherwise, assume one deployable application at the repository root.

If review confirms several deployable units, make a per-unit runbook before deployment:

- name each deployable unit, its context directory, runtime, port, and whether it needs a public URL;
- deploy each intended unit separately with its own `copas up --name` and `--context-dir`; never deploy the monorepo root as one application by default;
- order infrastructure/dependencies before the units that need them, then verify each public unit independently;
- call out Compose-only or non-HTTP worker topology as a current CLI boundary rather than guessing a deployment shape.

### 2. Choose defaults from evidence

Make the smallest reasonable set of decisions yourself:

- **Build:** keep the default Railpack builder. Choose Dockerfile only for a verified custom system dependency, unsupported stack, or intentionally hand-built image.
- **Path:** deploy the repository root unless inspection proves the app is in a subdirectory; use `--context-dir` for that subdirectory.
- **Port:** use the app’s documented/listening port; otherwise accept Copas’s default `3000`. The app must listen on `$PORT` and `0.0.0.0`.
- **Domain/TLS:** let Copas assign `<app>.<appsDomain>` and TLS when `copas info` reports an apps domain. Specify `--domain --tls` only when the user supplied a host.
- **Environment:** use `--env-file` and/or `--env`; mark sensitive supplied keys with `--secret`.
- **Storage:** add `--mount` and `--volume-size` only when inspection finds durable local state that belongs on a persistent volume.

### 3. Identify dependencies before deployment

Look for database/cache/queue evidence such as `DATABASE_URL`, `REDIS_URL`, ORM packages/configuration, migration directories, Compose services, or framework-specific database configuration.

- Tell the user what dependency was detected and which variable(s) the app expects.
- If startup runs migrations or requires the dependency, it must be reachable before deploying the app.
- For a supported managed engine, the planned infrastructure command is `copas db create <name> --project <project> --engine <engine> --deploy`. It provisions the database before the application and prints the in-cluster connection string. Run it only after the final confirmation for the full mutation plan.
- Treat the connection string as a secret: never place it in chat, a displayed command, a release record, a commit, or `--password`. Let Copas generate the database password by default.
- When local automation is approved, use `--json` to capture the connection string without printing it and write it only to a local, permission-restricted, gitignored env file for `copas up --env-file`; remove temporary secret files after deployment. If no safe handoff is available, report the dependency as blocked rather than exposing the value.
- Never invent database engine, host, credentials, or a successful migration.

### 4. Orient on the target Copas server

Authenticate and inspect the target when needed:

```bash
copas info
copas login --email you@example.com
copas project
```

`copas up --project <name>` creates the project if it does not yet exist. Use `copas project create <name>` only when project creation itself is the user’s requested task or should be completed before deployment.

If the CLI has no authenticated session, authentication is a blocker before deployment:

1. Ask the human for the email address they want to use **only if it was not already provided**. Do not infer, guess, or reuse an unrelated email address.
2. Run `copas login --email <user-email>`. This uses the CLI's configured default server; do not add `--server` unless the human explicitly asks to target a different Copas server.
3. Tell the human that Copas sent a one-time sign-in link to that inbox and that they must check their inbox (and spam/junk folder if needed) and open the link in a browser within five minutes.
4. The command waits for browser confirmation. Do not try to access their mailbox, request their password, run `copas up`, or claim login succeeded until the command finishes successfully.

If the user cannot receive or open the link, report authentication as blocked and stop before deployment.

### 5. Present one concise deploy summary, then ask for confirmation

Before database provisioning, source upload, build, or any production change, use this exact concise handoff format:

```text
Readiness: ready | blocked — <one-line reason>
Detected: <runtime>; <app/context>; <port>; <complexity/dependencies>
Changes: <database/infrastructure mutation, if any; otherwise none>
Plan: <ordered exact commands: dependency provisioning first when needed, then copas up; no secret values>
Needs: <only unresolved secret, database, DNS, or domain blockers; otherwise none>
Verification: <deployment status + public URL/health path to check>
Recovery: <next safe action or rollback plan>
```

Ask for **one final confirmation to execute the entire mutation plan**. Do not provision a database before that confirmation, and do not ask the user to choose routine defaults already supported by repository evidence.

### 6. Deploy with `copas up`

The standard path is:

```bash
copas up --project <project> --name <app> --path .
```

Use flags only when the plan requires them:

```bash
copas up --project <project> --name <app> \
  --context-dir <subdir> --port <port> \
  --env-file .env.production --env KEY=VALUE --secret SECRET_KEY \
  --mount <volume>:<path> --volume-size 1Gi \
  --domain <host> --tls
```

`copas up` follows the server-side build/deploy log by default. Re-running it with the same project and app name updates that application.

### 7. Verify the public endpoint

A successful deployment is not the last check. When `copas up` reports an assigned or requested public host, verify that the application responds from the best evidence-backed URL:

```bash
curl --fail --silent --show-error --location \
  --retry 5 --retry-all-errors --connect-timeout 5 --max-time 20 \
  https://<public-host><health-path> -o /dev/null
```

- Use the custom `--domain` host when one was supplied; otherwise use the auto-generated host printed by `copas up`.
- Use a health path found during repository inspection. If none is evidenced, fall back to `/`; do not guess an endpoint.
- Treat an HTTP 2xx or intentional redirect as reachable. Record the checked URL in the deployment summary.
- If the server has no apps domain and no custom domain was supplied, state that no public URL is configured; do not pretend that an external probe occurred.
- If the probe fails after a successful deployment, diagnose DNS/TLS, ingress, selected port, bind address, and application startup before declaring the release live.

Close every completed attempt with a release record: result (`live`, `blocked`, or `failed`), project/app, deployment ID, build path and context directory, dependency names/IDs (never connection strings or credentials), checked URL and outcome, plus the next recovery action when it is not live.

For a successful release, present this structured completion and always include the Copas Console link:

```text
Result: live
Project/app: <project>/<app>
Deployment: <deployment-id>
Checked URL: <public URL/health path> — <HTTP outcome>
Console: https://console.copas.sh/ — inspect the deployment and runtime state
```

Do not add a Console field to a `blocked` or `failed` result as a substitute for a successful deployment.

## JTBD command map

### Orient and organize

```bash
copas info                         # server URL, apps domain, registry, public URL
copas project                      # list projects in the active organization
copas project create <project>     # create a project explicitly
```

### Provision a managed database

```bash
# Engines: postgres, mysql, mariadb, mongo, redis.
copas db create <name> --project <project> --engine postgres --deploy
copas db list --project <project>
copas db get <database-id>
```

`create --deploy` provisions the database and returns its in-cluster connection string. For a two-step flow, use `copas db create` followed by `copas db deploy <database-id>`. When automation is approved, capture `--json` output locally without displaying the value; wire it into a permission-restricted, gitignored `--env-file` and mark the corresponding key with `--secret`. Never copy the connection string into a command shown to the human, chat, or release record.

### Ship or update source

```bash
copas up --project <project> --name <app>
```

Use this for first deploys and subsequent source updates. It is the CLI’s application create/update/build/deploy workflow; there is no separate build-only command.

### Check a release and its public URL

```bash
copas deployment list --project <project>
copas deployment status <deployment-id>
curl --fail --silent --show-error --location https://<public-host><health-path> -o /dev/null
```

`deployment status` streams the recorded build/deploy log until the deployment finishes. It is **not** a general runtime-log tail for an already running service. A successful deployment plus a successful public-URL probe is the condition for reporting the release as live.

### Recover a release

```bash
copas deployment redeploy <deployment-id>
```

This re-applies a previous deployment snapshot as a new deployment. State the deployment being restored and ask for confirmation before rolling back production.

### Maintain the client

```bash
copas update
copas update --check
copas version
```

## Failure navigation

Keep the failure diagnosis tied to the stage shown by the deployment log.

| Stage or signal | Check first | Usual next action |
| --- | --- | --- |
| Packing fails | `--path`, readable files, `.dockerignore`/`.gitignore`, ignored app files | Point `--path` at the app or correct the ignored/context files. Copas always excludes `.git`. |
| Railpack cannot detect runtime | manifest/lockfile exists in deploy context, monorepo root vs app directory | Set `--context-dir` to the app. Use Dockerfile only if Railpack remains unsuitable for evidenced reasons. |
| Dependency install/build fails | lockfile, runtime version, package/build script, system dependency | Fix source/config first; redeploy with `copas up`. Do not switch to Dockerfile merely to hide an unresolved dependency error. |
| No start command / exits immediately | start script/entrypoint, custom command, app path | Point to the correct context or use `--command` when the project has a known explicit start command. |
| Service cannot receive traffic | `--port`, `$PORT`, bind address | Ensure it listens on the selected port and `0.0.0.0`; then redeploy. |
| Database provisioning fails | engine/version, requested storage, deployment response | Report the exact error; do not guess a connection string or continue to `copas up` when the app requires the database. |
| Migration/config startup failure | required env names, database reachability, migration order | Provision/wire the dependency first, provide secrets safely, then redeploy. |
| Magic-link login times out or email is not received | entered email, inbox/spam folder, link expiry | Ask the human to retry the link flow; do not deploy until `copas login` succeeds. |
| TLS not ready | DNS record, requested host, public port 80 | Ensure the host resolves to the cluster and is reachable for ACME HTTP-01. |
| Pending/image pull failure | deployment status log and server/cluster availability | Report the exact error. This may require server/operator intervention. |

If the deployment succeeds but the running service later misbehaves, explain the CLI boundary: current CLI has deployment logs, not a direct runtime service-log command. Use the Web UI/API for runtime service inspection until a service-operations CLI command exists.

## Current CLI boundaries

Do not promise commands that do not exist. The current client CLI does **not** provide direct commands for:

- listing or inspecting individual services/apps;
- tailing runtime service logs, restart, scaling, or per-service rollback;
- managing Compose services; database operations are limited to `copas db create`, `deploy`, `list`, `get`, and `delete`;
- reading/editing env/secrets, domains, or project service health;
- selecting an organization other than the saved active/default organization.

When the user’s job needs one of these, explain the boundary and hand off to the Web UI/API. Do not tell the user to replace a source deployment with manual Docker/Kubernetes steps.

## Final deploy checklist

- [ ] Repository inspected; runtime, app directory, start behavior, and port are understood.
- [ ] Railpack selected unless concrete repository evidence requires Dockerfile.
- [ ] Database/cache/queue/storage needs are identified; required dependencies are ready.
- [ ] Env keys are supplied safely; sensitive keys are marked with `--secret`.
- [ ] The exact `copas up` command is ready and the user has confirmed it.
- [ ] Deployment log is followed; failure is diagnosed by stage.
- [ ] The reported public URL/health path responds successfully, or the absence of a public URL is stated explicitly.
- [ ] A release record states the deployment ID, checked URL, result, and next recovery action when needed.
- [ ] Production rollback is confirmed before `copas deployment redeploy`.
