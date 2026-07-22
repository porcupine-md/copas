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
- When it lists projects, show their names and ask: **“App ini mau di-deploy ke project yang mana?”** Use the user's choice, unless the user already named a project.

Use the selected project consistently in the repository plan, dependency provisioning, and every `copas up` command.

### 4. Review the repository

Inspect only the evidence needed to deploy:

- runtime and build markers (`package.json`, lockfiles, `go.mod`, `pyproject.toml`, and equivalent);
- start command, listening port, bind host, and an evidenced health endpoint;
- app directory and independently deployable units in a monorepo;
- environment templates, migrations, ORM configuration, and dependency variable names;
- durable files, workers, queues, caches, databases, and other dependencies.

Build a dependency map before choosing deployment commands. For every app or service, identify what it needs at startup, the repository evidence for that relationship, its environment-variable names, and whether the dependency already exists. Then create one serial runbook:

```text
managed database/cache → wait for its successful deployment → wire app environment
→ deploy its dependent API/worker → verify → deploy the next dependent service
```

Start with stateful dependencies, then deploy services that consume them, then their public-facing dependents. Keep all deploys sequential, including monorepo units, so a later service never races an unavailable dependency.

Choose defaults from that evidence:

- deploy source with `copas up` and Railpack by default;
- use the repository root unless the app is evidenced in a subdirectory;
- use the app's documented port, otherwise Copas's default `3000`; apps listen on `$PORT` and `0.0.0.0`;
- use the generated `<app>.<appsDomain>` host unless the user supplied a domain;
- choose Dockerfile only when the repository demonstrates that Railpack cannot build the application.

Summarize the runtime, app context, port, dependency order, and any genuinely missing input. Routine defaults need no questionnaire.

### 5. Ask once, then execute serially

Before creating infrastructure, uploading source, or deploying, present one concise plan and ask once for approval of the whole mutation plan. Name `copas up` as the **go-live deployment** action, rather than presenting it as an unexplained command:

```text
Detected: <runtime>; <app/context>; <port>; <dependencies>
Plan: <dependency 1 → deploy app 1 go live with copas up → dependency/app 2, in exact order>
Needs: <only unresolved email, secret, domain, or dependency input; otherwise none>
Verify: <public URL and evidenced health path>
```

For a simple application, say: **“Plan: deploy this app go live with `copas up` from the repository root; no dependencies or secrets; verify the public URL at `/`. Proceed with this go-live deployment?”**

After approval, complete every operation in dependency order. Finish one deployment before starting the next; this prevents infrastructure and service startup races.

## Dependencies before applications

When a repository needs a managed database, provision it first and wait for the command to finish successfully:

```bash
db_json="$(mktemp)"
copas db create <database> --project <project> --engine <engine> --deploy --json > "$db_json"
```

A successful `--deploy` response is the CLI's readiness contract for this flow. Capture its connection string from the private temporary file, map it to the variable found in the repository (for example, `DATABASE_URL`), and write it only to a permission-restricted, gitignored env file. Remove temporary secret files when they are no longer needed.

Then deploy the dependent application:

```bash
copas up --project <project> --name <app> --path . \
  --env-file <restricted-env-file> --secret <connection-variable>
```

For a monorepo or several services, use the same serial sequence for every unit:

```text
provision dependency → wait for success → wire its env → deploy dependent service → verify
```

Deploy database and application separately. Deploy all other services one at a time according to their dependency order. Keep connection strings in local restricted files, not in chat, displayed commands, commits, or release summaries.

If a dependency is already provisioned, inspect `copas db list` for its host and port. When its connection string is needed, capture `copas db get <database-id> --json` into a private temporary file using the same pattern above, then continue with the dependent service. If the repository does not establish a database engine or environment-variable mapping, ask for that one missing decision before provisioning.

## Deploy and verify

A normal source deployment is:

```bash
copas up --project <project> --name <app> --path .
```

Add only evidence-backed flags as needed:

```bash
copas up --project <project> --name <app> \
  --context-dir <subdir> --port <port> \
  --env-file <env-file> --env KEY=VALUE --secret SECRET_KEY \
  --mount <volume>:<path> --volume-size 1Gi \
  --domain <host> --tls
```

`copas up` follows the server-side build and deploy output. Once it succeeds and reports a public host, verify the health endpoint found during repository review; use `/` when none is declared:

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
| Build or start fails | Read the deployment output, correct the matching source/configuration issue, and run `copas up` again. |
| Dependency provisioning fails | Preserve the exact error, correct the dependency input, then resume from provisioning before the application deploy. |
| Application cannot receive traffic | Check selected port, `$PORT`, bind address, domain/TLS, and startup output; redeploy after correction. |
| Magic link expires | Run `copas login --email <email>` again and resume after it completes. |
| Public health probe fails | Check DNS/TLS, ingress, application startup, port, and bind address before reporting the release live. |

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
copas up --project <project> --name <app>       # source build + deploy
copas deployment list --project <project>       # deployment history
copas deployment status <deployment-id>         # recorded build/deploy output
copas deployment redeploy <deployment-id>       # confirmed rollback/redeploy
copas update                                    # client update
```

The CLI currently covers source deployments, managed databases, and deployment output. Use the Web UI/API for per-service runtime operations such as runtime-log tailing, scaling, restart, and service-level rollback.
