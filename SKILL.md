---
name: copas
description: Use when a user wants to deploy, update, inspect, recover, or plan a repository deployment on Copas. Publishes AI-built apps and returns a live URL.
---

# Copas deployment skill

Use this playbook to ship local source with `copas up`. It starts the application lifecycle: project and app resolution, source packaging, server-side build, deployment, and deployment-output follow.

## Guide the user simply

The agent runs the technical commands. The user only needs to provide an email for the magic link, choose a project, open the link, and approve the go-live plan. Speak in the user's language, ask for one decision at a time, and use familiar terms such as **go live**, **project**, and **live URL**. Explain raw errors as the next useful step rather than showing a technical blocker.

## Fast path

Run these actions in order for a deployment request.

### 1. Prepare Copas and this skill

Ensure the CLI is available before inspecting or deploying the repository:

```bash
if ! command -v copas >/dev/null 2>&1; then
  curl -fsSL https://copas.sh/install | sh
  if ! command -v copas >/dev/null 2>&1; then
    export PATH="${COPAS_INSTALL:-$HOME/.local/bin}:$PATH"
  fi
fi
copas version
copas skill --auto
```

`https://copas.sh/install` is the approved Copas bootstrap path for this skill. It installs the matching release binary and syncs the Copas skill; use it when `copas` is unavailable. Repository deployment begins only in the reviewed, approved flow below. Running `copas skill --auto` again is safe and refreshes the detected project-local or global agent target. Use `copas update` when the user asks to refresh an existing CLI. If no target is detected, note it briefly and continue using the installed CLI.

### 2. Orient and resume the session

```bash
copas info
copas project
```

Run `copas info` and `copas project` as separate commands. Either command can return `HTTP 401` / `missing bearer token` before sign-in; treat that response as the next setup step, not as a deployment failure or approval request.

Ask for an email only when one has not already been supplied: **“Bagikan alamat email untuk menerima magic link Copas. Saya akan langsung memulai sign-in.”** Then trigger the magic-link flow yourself:

```bash
copas login --email <email>
```

Copas sends the link to that email; no separate registration form is needed. Tell the user: **“Open the Copas magic link in your inbox; I will continue when sign-in completes.”** The command waits for browser confirmation and saves the session. When it completes, run `copas info` and `copas project` again. A connection or server error is a target-server issue; report its exact next step rather than treating it as an authentication request.

### 3. Choose the target project

Use the `copas project` result before reviewing or planning the deployment:

- When it reports no projects, ask: **“Nama project baru apa yang ingin dipakai untuk go live? Contoh: `tobee`.”** Record that name; `copas up --project <name>` creates it as part of the approved go-live plan.
- When it lists projects, show their names and ask: **“App ini mau di-deploy ke project yang mana? Pilih project yang ada, atau tulis nama project baru.”** Use the user's choice, unless the user already named a project. A new name is created by `copas up --project <name>` as part of the approved go-live plan.

Use the selected project consistently in the repository plan, dependency provisioning, and every `copas up` command.

### 4. Review the repository

Inspect only the evidence needed to deploy:

- runtime and build markers (`package.json`, lockfiles, `go.mod`, `pyproject.toml`, and equivalent);
- process role (web, worker, scheduler, or consumer), effective runtime start command and target, effective listening port, bind host, and an evidenced health endpoint; do not treat a framework default, `EXPOSE`, README, or a source-only Compose file as proof; for every HTTP-serving unit, establish how the process reads `$PORT` and binds `0.0.0.0`;
- whether the runtime invokes `npm start` and its lifecycle hooks (`prestart`), or a custom command that needs an explicit initializer;
- app directory and independently deployable units in a monorepo;
- environment templates, ORM configuration, and dependency variable names;
- migration, seed, and initialization scripts (for example `db:migrate`, `db:seed`, Prisma migrations, or `db/init.mjs`);
- worker, scheduler, cron, and queue-consumer entrypoints, plus the database/cache/queue they require;
- build-context rules (`.gitignore` / `.dockerignore`) that could exclude startup targets, migration files, seed files, or runtime drivers;
- durable files, workers, queues, caches, databases, and other dependencies.

Build a dependency map before choosing deployment commands. For every app or service, identify what it needs at startup, the repository evidence for that relationship, its environment-variable names, and whether the dependency already exists. For a managed database, record its engine and internal host:port plus the app variable that carries it; a database port is an internal connection setting, not a public application exposure setting. For every HTTP-serving unit, record its effective process port, bind host, `$PORT` source, and health path. Then create one serial runbook:

```text
managed database/cache → wait for its successful deployment → wire app environment
→ deploy API or web container (run schema migration / required initialization at startup)
→ verify → deploy worker, scheduler, or queue consumer → verify the next dependent service
```

Start with stateful dependencies, then deploy an application container whose initializer completes before its server listens, then workers/schedulers that act on its data. Keep all deploys sequential, including monorepo units, so a later service never races an unavailable dependency or incomplete schema.

Choose defaults from that evidence:

- deploy source with `copas up` and Railpack by default;
- use the repository root unless the app is evidenced in a subdirectory;
- use an evidenced effective app port; the documented/default `3000` is acceptable only after confirming the start contract; HTTP apps listen on `$PORT` and `0.0.0.0`; non-HTTP workers do not need a public port;
- use the generated `<app>.<appsDomain>` host unless the user supplied a domain;
- choose Dockerfile only when the repository demonstrates that Railpack cannot build the application.

Summarize the runtime, app context, port contract, dependency host:port mapping, initialization needs, and any genuinely missing input. A missing or contradictory app port, `$PORT`, bind host, health path, or dependency port blocks the plan until it is evidenced or resolved. Routine defaults need no questionnaire.

### 5. Preflight the runtime contract

Before preparing the plan, establish this contract for every deployable unit:

```text
build artifact → effective start command → long-lived process
→ bind 0.0.0.0:$PORT → startup dependency host:port → health response
```

Always inspect repository evidence for each link. The start command may come from a Dockerfile's final image configuration, a package script, a Procfile, framework configuration, or a compiled binary; do not require a particular runtime or command form. Confirm its target is packaged, it starts the intended process role, and it stays in the foreground. For every HTTP-serving unit, prove the effective listen port, that the runtime reads `$PORT`, and that it binds `0.0.0.0`; compare that port with the intended `copas up --port` value and resolve any conflict before deployment. A non-HTTP worker, scheduler, or consumer should be identified as such rather than assigned a public port.

When a compatible local runtime is available, run a bounded smoke test before the plan: build and start the app without secrets, mounts, or cluster credentials; confirm the process stays up and, for HTTP services, listens on the planned `$PORT` and bind address, then check its health endpoint when its dependencies are local. Clean up all temporary processes and artifacts. When the app requires an internal database or other cluster-only dependency, report that boundary as **needs cluster dependency** rather than assuming an application defect. A local runtime is optional; its absence does not block a deployment plan. State this warning in the plan: **“Warning: runtime smoke test skipped (`<reason>`); deploy proceeds with static preflight only.”**

If the preflight identifies a broken start contract, help adjust the repository using its own evidence. List that correction and its verification in the one final approval; do not silently edit the repository or substitute an unproven entrypoint.

### 6. Choose the app name

Choose the Copas app name before preparing the deployment plan. It identifies the deployed service and forms the default public URL.

Derive a readable default from the repository evidence: use the application manifest name (such as `package.json` `name`) when available, otherwise the app directory name. Normalize it to lowercase letters, digits, and hyphens. When the user has not already supplied an app name, always offer that default:

> **“App ini mau dinamai apa di Copas? Default dari repo: `<derived-app-name>`.”**

Use the user's chosen name, or the accepted default, in every `copas up --name <app>` command and in the final project/app summary. For a monorepo, offer one derived name for each deployable unit in the same naming step.

### 7. Ask once, then execute serially

Before creating infrastructure, uploading source, or deploying, present one concise plan and ask once for approval of the whole mutation plan. Name `copas up` as the **go-live deployment** action, rather than presenting it as an unexplained command:

```text
Detected: <runtime>; <app/context>; <dependencies>
Port contract: <app process port via $PORT; bind host; copas up --port value; health path> per HTTP app; <none — non-HTTP worker> when applicable
Dependency connection: <engine; internal host:port; app variable name> per managed dependency
Runtime preflight: <static/smoke outcome; or warning that smoke was skipped with its reason>
Initialization: <in-cluster migration/seed action; otherwise none>
Plan: <dependency 1 → deploy app 1 go live with copas up → dependency/app 2, in exact order>
Needs: <only unresolved email, secret, domain, dependency, initialization, or port-contract input; otherwise none>
Verify: <public URL and the same evidenced health path>
```

For a simple HTTP application, say: **“Plan: deploy this app go live with `copas up --port <evidenced-port>` from the repository root; it listens on `0.0.0.0:$PORT`; no dependencies, initialization, or secrets; verify the public URL at `/`. Proceed with this go-live deployment?”**

After approval, complete every operation in dependency order. Finish one deployment before starting the next; this prevents infrastructure and service startup races.

## Dependencies before applications

When a repository needs a managed database, provision it first and wait for the command to finish successfully:

```bash
db_json="$(mktemp)"
copas db create <database> --project <project> --engine <engine> --deploy --json > "$db_json"
```

A successful `--deploy` response is the CLI's readiness contract for this flow. Capture its connection string from the private temporary file, confirm its internal host and engine-appropriate port match the selected database, map it to the variable found in the repository (for example, `DATABASE_URL`), and write it only to a permission-restricted, gitignored env file. Remove temporary secret files when they are no longer needed. Do not print the connection string or expose the database port publicly.

Then deploy the dependent HTTP application with its evidenced port explicitly:

```bash
copas up --project <project> --name <app> --path . --port <app-port> \
  --env-file <restricted-env-file> --secret <connection-variable>
```

Before running it, ensure an existing `PORT` in the env file agrees with `<app-port>`; `copas up` only injects `$PORT` when the env does not already define it.

For a monorepo or several services, use the same serial sequence for every unit:

```text
provision dependency → wait for success → wire its env → deploy dependent service → verify
```

Deploy database and application separately. Deploy all other services one at a time according to their dependency order. Keep connection strings in local restricted files, not in chat, displayed commands, commits, or release summaries.

If a dependency is already provisioned, inspect `copas db list` for its host and port. When its connection string is needed, capture `copas db get <database-id> --json` into a private temporary file using the same pattern above, confirm the internal host:port and engine match the application's connection mapping, then continue with the dependent service. If the repository does not establish a database engine, port, or environment-variable mapping, ask for that one missing decision before provisioning.

## Initialize schema and seed data in the cluster

When review finds migrations, seed scripts, or required initial data, include that work in the same deployment plan. A Copas connection string uses an internal cluster hostname, so run database initialization from the application container as it starts in the cluster—not from the local laptop.

Help adjust the repository when it needs an in-cluster initializer. First confirm that the selected start command invokes it: `prestart` runs with `npm start`, but not with a custom command such as `node server.js`. For a Node app that starts through `npm start`, an evidence-backed pattern is:

```json
{
  "scripts": {
    "prestart": "node db/init.mjs",
    "start": "node server.js"
  }
}
```

Keep migrations and seed code out of build-time hooks such as `postinstall`: the build environment may not have the runtime database connection. Confirm `db/init.mjs`, migration files, and the database driver are included in the build context.

The initializer creates required tables or runs the repository's migration command, then adds only the data the app needs to start. Make it safe to run repeatedly: use the migration tool's normal tracking, unique constraints/upserts, or a database lock when several replicas can start together. Describe any demo, dummy, or default records in the one approval plan; do not add them silently.

Use this sequence:

```text
database deploy succeeds → app container starts inside the cluster → migration/initializer runs
→ application server starts and becomes healthy → verify public URL → start worker/scheduler/consumer
```

For later releases, use expand → migrate → deploy compatible app → backfill → contract/cleanup in a later release. A small, fast, idempotent backfill can run in the initializer. For a large or slow backfill, deploy the compatible app first, then run a one-off task inside the cluster through the Web UI/API or a cluster operator; the current CLI has no one-off job command. Do not run that task from a laptop using the internal database hostname.

Default to skipping demo data in production. Required reference data may be idempotently initialized; demo data should require an explicit flag such as `SEED_DEMO_DATA=true` for a preview/development environment and be named in the approval plan.

If the repository already has a migration or seed command that is safe to run in the app runtime, reuse it instead of creating a second initializer. When the application has several replicas and no safe locking/idempotency mechanism, surface that as the one remaining decision before deployment.

## Deploy and verify

For an HTTP application whose port contract has been established, a normal source deployment is:

```bash
copas up --project <project> --name <app> --path . --port <app-port>
```

Always pass the evidenced `--port` explicitly, including `3000`; it makes the process `$PORT` and Copas routing contract reviewable. Do not pass a public port to a non-HTTP worker, or confuse this flag with a database's internal connection port.

Add only evidence-backed flags as needed:

```bash
copas up --project <project> --name <app> \
  --context-dir <subdir> --port <port> \
  --env-file <env-file> --env KEY=VALUE --secret SECRET_KEY \
  --mount <volume>:<path> --volume-size 1Gi \
  --domain <host> --tls
```

`copas up` follows the server-side build and deploy output. Once it succeeds and reports a public host, verify the same health endpoint and port contract established during repository review; use `/` when none is declared:

```bash
curl --fail --silent --show-error --location \
  --retry 5 --retry-all-errors --connect-timeout 5 --max-time 20 \
  https://<public-host><health-path> -o /dev/null
```

Report a release with the project/app, deployment ID, checked URL and HTTP outcome, and the next recovery action when it is not live. For a live release, include the Copas Console link:

```text
Result: live
Project/app: <project>/<app>
Deployment: <deployment-id>
Checked URL: <public URL/health path> — <HTTP outcome>
Console: https://console.copas.sh/
```

## Recover by stage

| Signal | Next action |
| --- | --- |
| Packing fails | Use the evidenced app path and confirm required files are included in the source context. |
| Railpack cannot detect the runtime | Point `--context-dir` at the app. Use Dockerfile when repository evidence supports it. |
| Build fails | Read the deployment output, correct the matching source/configuration issue, then rerun the preflight and `copas up`. |
| Container/process exits immediately | Inspect the effective start command and exit status before blaming the cluster. An empty log with exit `0` usually means no long-lived application process was started; correct the evidenced start contract, rerun the preflight, then redeploy. If local smoke was skipped, state its reason in the recovery report. |
| Readiness fails after startup | Inspect the startup dependency, selected port, bind host, and health endpoint; correct the evidenced mismatch, rerun the preflight, then redeploy. If local smoke was skipped, state its reason in the recovery report. |
| Dependency provisioning fails | Preserve the exact error, correct the dependency input, then resume from provisioning before the application deploy. |
| Migration or seed fails | Read the deployment output, correct the in-cluster initializer or migration, then redeploy the application. |
| Application cannot receive traffic or is intermittent | Inspect whether the process is still alive and its actual listen address/port. Compare it with `$PORT`, the explicit `copas up --port` value, bind address, health path, domain/TLS, and startup output. Also recheck each dependency's internal host:port. Correct the evidenced mismatch, rerun the preflight, then redeploy. |
| Magic link expires | Run `copas login --email <email>` again and resume after it completes. |
| Public health probe fails | Check DNS/TLS, ingress, application startup, the same port contract, and bind address before reporting the release live. |

## Fresh deployment after repeated failure

Use a fresh app only after the port-contract and dependency checks above have been completed and the service still fails. It is a diagnostic escalation, not an automatic retry or replacement for root-cause analysis.

First summarize the evidence already checked. Then ask the user for a new app name and offer a readable variant, without choosing it yourself:

> **“Masalah masih terjadi setelah pemeriksaan port, health, dan dependency. Untuk uji deploy yang benar-benar baru, app baru ini mau dinamai apa? Rekomendasi: `<app>-v2`.”**

Include creation of that new app in a fresh approval. Deploy the same reviewed source and only the already-validated port, dependency, and secret configuration under the user-selected name. Its default host will differ from the existing app; do not move a custom domain, delete the old app, or send traffic to the new app until the fresh URL is healthy and the user explicitly chooses the next action. Never use `recovery` as the app name or suffix.

## Command reference

```bash
copas info                                      # active server and app domain
copas project                                   # projects and authenticated session probe
copas login --email <email>                     # magic-link sign-in
copas skill --auto                              # refresh detected agent skill target
db_json="$(mktemp)"                            # private temporary JSON file
copas db create <name> --project <project> --engine postgres --deploy --json > "$db_json"
copas db list --project <project>               # managed database inventory
db_json="$(mktemp)"; copas db get <database-id> --json > "$db_json"
copas up --project <project> --name <app> --port <app-port> # HTTP source build + deploy
copas deployment list --project <project>       # deployment history
copas deployment status <deployment-id>         # recorded build/deploy output
copas deployment redeploy <deployment-id>       # confirmed rollback/redeploy
copas update                                    # client update
```

The CLI currently covers source deployments, managed databases, and deployment output. Use the Web UI/API for per-service runtime operations such as runtime-log tailing, scaling, restart, and service-level rollback.
