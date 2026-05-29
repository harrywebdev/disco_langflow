# Self-hosted Langflow on Disco.cloud

This repo deploys [Langflow](https://www.langflow.org/) (the open-source LLM workflow builder by DataStax) to a private server via [Disco.cloud](https://disco.cloud), a self-hosted, open-source PaaS that runs on any Ubuntu 24.04 box you own.

- **Target server:** `server.domena.cz`
- **Public domain:** `lf.domena.cz`
- **Image:** [`langflowai/langflow:1.9.5`](https://hub.docker.com/r/langflowai/langflow)
- **Repo (deploy source):** `harrywebdev/disco_langflow`

The repo only contains the deploy contract — `disco.json` plus this guide. No Dockerfile: Disco pulls the official Langflow image straight from Docker Hub. The architecture intentionally diverges from [`datastax/host-langflow`](https://github.com/datastax/host-langflow), which is a hello-world starter and ships **no persistent volume**, **no Postgres**, and **no auth** — fine for a demo, unsafe for production.

---

## What gets deployed

A single Disco project, `langflow`, with one `web` service running the official Langflow image and one volume for persistent data. Postgres is provided by Disco's built-in addon and attached to the project as `LANGFLOW_DATABASE_URL`. Caddy (built into Disco) terminates TLS and routes `lf.domena.cz` → port 7860 inside the container.

```
         Internet
            │ 443
            ▼
   ┌─────────────────┐
   │ Caddy (Disco)   │  Let's Encrypt TLS, automatic
   └────────┬────────┘
            │ HTTP
            ▼
   ┌─────────────────┐         ┌────────────────┐
   │ langflow:7860   │ ──────► │ Disco Postgres │
   │ (web service)   │         │ (addon)        │
   └────────┬────────┘         └────────────────┘
            │
            ▼
   /app/langflow  (volume: langflow-data)
   logs, file uploads, monitor data, secret keys
```

---

## Prerequisites

| | |
|---|---|
| Server | Ubuntu 24.04 LTS, root SSH access, ports `22`, `80`, `443` reachable |
| DNS | `lf.domena.cz` → server's public IP (A record) |
| Local | [Disco CLI](https://disco.cloud/docs/cli/install/) installed: `curl https://cli-assets.letsdisco.dev/install.sh \| sh` |
| GitHub | Push access to `harrywebdev/disco_langflow` |

---

## One-time server setup

If `server.domena.cz` is already running Disco, skip to the next section.

```bash
disco init root@server.domena.cz
```

This SSHes in, installs Docker + the disco-daemon + Caddy, opens 80/443, and prints an invite URL ending in `dashboard.disco.cloud/i/...`. Open it in a browser and finish onboarding via GitHub OAuth.

Then create a self-hosted **GitHub App** from the dashboard ("Set up GitHub integration"). Disco names it `Disco server.domena.cz` by default. Install it on your account/org and grant access to the `harrywebdev/disco_langflow` repo. Webhooks then land directly on your server — Disco's central infrastructure never sees app traffic.

---

## Deploying Langflow

The order matters: the project must exist before env vars or Postgres can be attached.

### 1. Add the project

```bash
disco projects:add \
  --name langflow \
  --domain lf.domena.cz \
  --github harrywebdev/disco_langflow
```

Caddy will start provisioning a Let's Encrypt cert for `lf.domena.cz` as soon as DNS resolves to the server.

### 2. Provision Postgres (via Disco's addon)

```bash
# Install the Postgres addon on the server (once per server)
disco postgres:addon:install

# Spin up a Postgres 16 instance (once per server, shared by projects)
disco postgres:instances:add --image postgres --version 16

# Create a database on that instance (name is auto-generated and printed on success)
disco postgres:databases:add --instance <instance-name-from-previous-step>

# Attach the database to the project and inject the URL as LANGFLOW_DATABASE_URL
disco postgres:databases:attach \
  --instance <instance-name-from-previous-step> \
  --database <database-name-from-previous-step> \
  --project langflow \
  --env-var LANGFLOW_DATABASE_URL
```

`disco postgres:instances:add` and `disco postgres:databases:add` each print the name they generated on success — pass those back in as `--instance` and `--database`. The `--env-var` flag overrides Disco's default (`DATABASE_URL`) so Langflow picks the URL up under the name it actually reads.

### 3. Set the rest of the env

`disco.json` deliberately contains **no** env vars — that's a Disco rule. Set them via the CLI:

```bash
# Where Langflow keeps logs, file uploads, monitor data, AND the encryption key.
# Must match the volume's destinationPath in disco.json.
disco env:set --project langflow LANGFLOW_CONFIG_DIR=/app/langflow

# Turn off the dev-mode "anyone is admin" behavior.
disco env:set --project langflow LANGFLOW_AUTO_LOGIN=false

# Initial superuser, created on first boot.
disco env:set --project langflow LANGFLOW_SUPERUSER=admin LANGFLOW_SUPERUSER_PASSWORD="$(openssl rand -base64 24)"

# Encrypts credentials/API keys stored in the DB. If this ever changes,
# every encrypted value becomes unreadable — store it somewhere safe.
disco env:set --project langflow LANGFLOW_SECRET_KEY="$(openssl rand -base64 32)"

# New signups require admin approval instead of activating immediately.
disco env:set --project langflow LANGFLOW_NEW_USER_IS_ACTIVE=false
```

Record the generated `SUPERUSER_PASSWORD` and `SECRET_KEY` in a password manager before moving on.

### 4. Deploy

A push to `main` triggers a build automatically via the GitHub App webhook. For a manual first deploy:

```bash
disco deploy --project langflow
```

Follow the build:

```bash
disco logs --project langflow
```

Once Caddy has the cert, `https://lf.domena.cz` serves the Langflow UI. Log in as `admin` with the password from step 3.

---

## Operations

```bash
# Tail logs
disco logs --project langflow --service web

# Open a shell / run one-off commands inside the container
disco run --project langflow "bash"

# Backup the data volume (logs, secret key, uploads)
disco volumes:export --project langflow --volume langflow-data --output langflow-data.tar.gz

# Restore on a different server
disco volumes:import --project langflow --volume langflow-data --input langflow-data.tar.gz

# Direct Postgres access via tunnel
disco postgres:tunnel --project langflow

# Update env vars (re-deploys automatically)
disco env:set --project langflow KEY=VALUE
disco env:list --project langflow

# Scale (single-server: keep at 1; Langflow's worker model is gunicorn-style inside the container)
disco scale:set --project langflow web=1
```

### Upgrading Langflow

Bump the image tag in `disco.json`:

```diff
-      "image": "langflowai/langflow:1.9.5",
+      "image": "langflowai/langflow:1.10.0",
```

Commit, push, done — Disco does a blue/green swap via Docker Swarm with no downtime.

Always pin a version. Avoid `:latest` in production: it makes rollbacks ambiguous and rebuilds can pick up breaking changes silently.

### Backups

Two things to back up regularly:

1. **Postgres** — via `disco postgres:tunnel` + `pg_dump`, or your own cron job.
2. **`langflow-data` volume** — `disco volumes:export ...`. This is where the `LANGFLOW_SECRET_KEY` cache, monitor DB, and uploaded files live.

---

## Why this shape

A few choices worth explaining since the host-langflow repo nudges you in a different direction:

- **Official image, not a custom Dockerfile.** Langflow's own `langflowai/langflow` image already runs `langflow run --host 0.0.0.0 --port 7860` as its CMD. Wrapping it in our own Dockerfile (like `datastax/host-langflow` does) buys nothing and adds a build step.
- **Postgres via Disco addon, not a sidecar container.** Disco's addon manages a single Postgres instance shared across projects on the server — one place to back up, one place to upgrade, and the URL is wired in via `--env-var` so Langflow doesn't need to know it's there.
- **Volume at `/app/langflow`.** This is `LANGFLOW_CONFIG_DIR`. Without it you lose the auto-generated encryption key on every container rebuild, which silently breaks every stored credential. The host-langflow `bm/` compose omits this — don't follow that pattern.
- **`AUTO_LOGIN=false` + superuser + secret key.** Production posture. Without these, anyone hitting the URL becomes admin.
- **No `LANGFLOW_HOST` / `LANGFLOW_PORT` overrides.** The image already binds `0.0.0.0:7860` by default; setting them is redundant noise.

---

## Useful links

- Disco docs: https://disco.cloud/docs/
- `disco.json` reference: https://disco.cloud/docs/disco-json/
- Langflow docker example (upstream): https://github.com/langflow-ai/langflow/tree/main/docker_example
- Langflow env var reference: https://docs.langflow.org/environment-variables
- Disco Pocketbase example (volume pattern this repo follows): https://github.com/letsdiscodev/example-pocketbase
