# NeuralOps Nexus

**A self-hosted team workspace where your people and AI personas with real tools work side by side.**

![NeuralOps Nexus in action: an AI persona works with MCP tools and nothing is saved until a person clicks Allow](docs/demo.gif)

`@mention` a persona in any channel like a colleague. It answers with a plain LLM, or it uses real tools through MCP servers (your code graph, files, the web, GitHub, Jira, Postgres and more), with approval gates you control. It all runs on your server, with your data and your own model keys.

> **Licence:** NeuralOps Nexus is free to self-host. It ships as a compiled Docker image; the source code is not public yet. We plan to publish it under a source-available licence as the project grows. Our MCP servers [contextgraph](https://github.com/mapax-io/contextgraph) and csv-mcp are open source (Apache-2.0).

This repository holds **only the installer files**: `install.sh`, the compose file, the nginx config and the `neuralops` command. NeuralOps itself ships as the Docker image [`noamanfaisal/neuralops`](https://hub.docker.com/r/noamanfaisal/neuralops).

---

## Contents

1. [Quick install](#quick-install)
2. [Requirements](#requirements)
3. [How your team reaches the server](#how-your-team-reaches-the-server)
4. [What the installer does](#what-the-installer-does)
5. [After installing: claim your server](#after-installing-claim-your-server)
6. [First steps in the app](#first-steps-in-the-app)
7. [The `neuralops` command](#the-neuralops-command)
8. [Updating, backups and restore](#updating-backups-and-restore)
9. [Installer settings (unattended installs)](#installer-settings-unattended-installs)
10. [Manual install (advanced)](#manual-install-advanced)
11. [Privacy: what leaves your server](#privacy-what-leaves-your-server)
12. [Troubleshooting](#troubleshooting)
13. [Uninstall](#uninstall)
14. [Help and community](#help-and-community)

---

## Quick install

On a Linux server:

```bash
curl -fsSL https://raw.githubusercontent.com/noamanfaisal/neuralops-get/main/install.sh | bash
```

Prefer to read it first? Download, inspect, then run:

```bash
curl -fsSL https://raw.githubusercontent.com/noamanfaisal/neuralops-get/main/install.sh -o install.sh
less install.sh
bash install.sh
```

When it finishes, it prints your server's address. Open [neuralopsnexus.ai](https://neuralopsnexus.ai), add that address, and claim the server (see [below](#after-installing-claim-your-server)).

---

## Requirements

| | Minimum |
|---|---|
| Operating system | Linux. Tested on Ubuntu 22.04 / 24.04 and Debian 12 |
| CPU | x86_64 (Intel/AMD). ARM servers are not supported yet |
| Memory | 4 GB (the installer warns below ~3.5 GB) |
| Disk | 15 GB free (the image is a few GB; your data grows over time) |
| Docker | Docker with Compose v2. Installed for you if missing (the installer asks first) |
| Network | Public-IP mode: ports **80 and 443** open. Tailscale mode: a free Tailscale account |

Not supported: 32-bit machines, and container-only hosts where Docker can't run inside (for example RunPod pods).

---

## How your team reaches the server

The installer asks **one question**:

| Option | Best for | What you get |
|---|---|---|
| **1. This server has a public IP** (default) | A cloud server: AWS, Azure, Google Cloud, DigitalOcean, Hetzner, Contabo… | A free random HTTPS address `<xxxxxx>.on.neuralopsnexus.ai`, certificate by Caddy. Ports 80 and 443 must be open. Updates and restarts keep the address. If the server's IP changes, the `neuralops` command moves the address to the new IP (checked at boot and every 6 hours). |
| **2. Tailscale** | A machine at home or in the office, behind a router | A permanent HTTPS address `https://<machine>.<tailnet>.ts.net` through Tailscale Funnel. Teammates install nothing. Use `NEURALOPS_TS_REACH=serve` to make it reachable only from your own Tailscale devices. |
| **Your own domain** (flag) | You already have a domain | `--domain team.example.com`. Point the domain's DNS (an A record) at the server first; Caddy fetches and renews the certificate. |

---

## What the installer does

Every step prints one line (✔ done, ! warning, ✘ stopped with the reason). Running it again is safe: a finished install stops with "already installed", and a half-finished one resumes where it stopped.

1. **Checks the machine:** Linux, x86_64, memory, free disk.
2. **Docker:** installs it from Docker's official script if missing (asks first).
3. **Asks how the server is reached** (above), and sets up the address or Tailscale.
4. **Prepares the files** in `/opt/neuralops`: downloads the compose file and nginx config, writes the settings files, and **generates this install's secrets once**, stored with mode 600. They are never regenerated, because that would replace your live database password.
5. **Starts everything:** creates the vector database, runs the database setup (migrate, permissions, model/MCP catalog), and waits until the server answers.
6. **Installs the `neuralops` command** and prints your address and next steps.

**What ends up on your server:**

```
/opt/neuralops/
├── docker-compose.neuralops.yaml
├── neuralops/
│   ├── infra.env        image version, database names, address settings
│   ├── app.env          this server's public address
│   ├── secrets.env      generated once, mode 600, never regenerated
│   ├── install.conf     settings the neuralops command reads
│   └── neuralops-cli.sh the neuralops command (/usr/local/bin/neuralops links here)
├── projects/            files your personas create (owned by UID 1000)
└── backups/             output of `neuralops backup` (mode 700)
```

Your data lives in Docker volumes that survive updates: the Postgres database (including the pgvector store), Redis, the embedding model cache and, for HTTPS installs, Caddy's certificates.

**Services that run:**

| Service | What it does |
|---|---|
| `nucleus` | The orchestrator: sign-in, rights, workspace, chat, schedules, chains, REST API and MCP gateway |
| `nucleus-celery`, `nucleus-celery-beat` | Background jobs and the scheduler |
| `nexus-ai` | The AI engine: runs models, builds prompts and toolsets, calls MCP tools, embeddings and search |
| `centrifugo` | Real-time message delivery to the browser |
| `postgres` | Your data and the vector store (pgvector), one database for both |
| `redis` | Job queue and real-time engine |
| `nginx` | The single entry point |
| `caddy` | HTTPS certificates (public-IP and domain installs) |

The database and Redis listen only on the machine itself (`127.0.0.1`), never on the internet.

---

## After installing: claim your server

1. Open [neuralopsnexus.ai](https://neuralopsnexus.ai) and sign in (or create an account).
2. **Add a server** and paste the address the installer printed.
3. The app shows a short code. Type it at the portal to **claim the server**. You become its owner.

There is no frontend to run: the web app is hosted, and it connects to **your** server.

---

## First steps in the app

Type `/` in the message box to see every command, and `@` to mention a persona.

| Step | Command | What you set |
|---|---|---|
| Add a model | `/add-model` | Provider (OpenAI, Anthropic, Google, DeepSeek, Groq, Mistral, Ollama, or any OpenAI-compatible endpoint), the model id, and your key (stored encrypted). **Check** verifies it before saving. Free-tier presets are in the catalog. |
| Add tools | `/add-mcp`, or pick from the catalog | An MCP server (GitHub, Jira/Confluence, Postgres, Slack, Google Drive…). Connect with one-click OAuth where supported, a static token, or your own OAuth app. |
| Add a persona | `/add-persona` | The name you'll `@mention`, its model, its MCP servers, its system prompt, and a level per tool: **auto**, **ask** (pauses for your approval) or **off**. |
| Talk | `@Persona your question` | Shape the reply with `@chart`, `@table`, `@diagram`. Stop or nudge a reply while it runs. |
| Automate | Routines, schedules, chains, inbound hooks | Saved methods, work on a timer, multi-step chains with a supervisor persona, and URLs other systems can call. |
| Invite | `/invite someone@example.com` | Add a person to a topic or a whole project. |

**Guardrails built in:** approval cards for risky tool calls, per-server repository allow-lists the engine enforces, a full AI audit log, roles and per-object permissions, and field-level encryption for stored keys.

**Use your server as a tool:** mint a gateway token and point Claude Desktop, Cursor or any MCP client at `https://<your-server>/api/v1/mcp/`. It works with exactly your permissions.

---

## The `neuralops` command

| Command | What it does |
|---|---|
| `neuralops status` | What's running, the address, the version |
| `neuralops address` | The current address (re-syncs a renamed Tailscale machine or a changed public IP) |
| `neuralops update [version]` | Backs up first, then upgrades (default: latest) and runs the database setup |
| `neuralops backup` | Database, secrets and project files into `backups/` |
| `neuralops restore <folder>` | Puts a backup back (secrets only onto a fresh install, or with `--use-backup-secrets`) |
| `neuralops logs [service]` | Follows the logs, for everything or one service (`nucleus`, `nexus-ai`…) |
| `neuralops restart` / `stop` / `start` | As named |
| `neuralops uninstall` | Removes the containers; your data is kept unless you type `DELETE` |

---

## Updating, backups and restore

```bash
neuralops update          # backs up, then moves to the latest release
neuralops update 1.1.8    # or a specific version
```

`neuralops backup` writes three things into `backups/neuralops-<date-time>/`: `database.sql.gz` (both databases and roles), `secrets.env`, and `projects.tar.gz`.

> **Keep backups private.** A backup includes `secrets.env`. Your stored model keys are encrypted with it, so a backup without it can't decrypt them, and anyone holding it can.

**Upgrade gotchas** (each found on a real install):

- Never run `docker compose down -v`. Your Postgres volume keeps the password it was created with; deleting it deletes your data.
- Never re-run `init-secrets` on an existing install.
- If you manage `infra.env` by hand, keep `NEURALOPS_IMAGE` in step with the version you want, or an old pinned tag keeps running.

---

## Installer settings (unattended installs)

Set these as environment variables (or the flags shown) before `bash`:

| Setting | Default | Meaning |
|---|---|---|
| `NEURALOPS_MODE` | `public` | `public`, `tailscale`, or `domain` |
| `NEURALOPS_DOMAIN` / `--domain` | — | Your own domain, for `domain` mode |
| `NEURALOPS_PUBLIC_IP` | auto | Only when the server's public IP differs from the one it reaches the internet from (e.g. behind NAT) |
| `NEURALOPS_TS_REACH` | `funnel` | `funnel` (anyone with the link) or `serve` (only your Tailscale devices) |
| `NEURALOPS_TS_AUTHKEY` | — | A Tailscale auth key, for unattended Tailscale installs |
| `NEURALOPS_TS_HOSTNAME` | `neuralops` | The Tailscale machine name |
| `NEURALOPS_YES` / `-y` | — | `1` = never ask, take every default |
| `NEURALOPS_VERSION` | current release | Which release to install |
| `NEURALOPS_DIR` | `/opt/neuralops` | Install folder |

Example:

```bash
curl -fsSL https://raw.githubusercontent.com/noamanfaisal/neuralops-get/main/install.sh \
  | NEURALOPS_YES=1 NEURALOPS_MODE=domain NEURALOPS_DOMAIN=team.example.com bash
```

---

## Manual install (advanced)

If you'd rather wire it yourself (your own reverse proxy, load balancer or orchestration):

1. Download `docker-compose.neuralops.yaml` and `neuralops/nginx.conf` from this repository, plus the env templates.
2. Generate this install's secrets **once**:
   ```bash
   mkdir -p neuralops projects
   docker run --rm noamanfaisal/neuralops:latest init-secrets > neuralops/secrets.env
   chmod 600 neuralops/secrets.env
   sudo chown -R 1000:1000 ./projects
   ```
3. Set your public address in `neuralops/app.env` (`NEURALOPS_SERVER_URL=https://…`, same value for `CENTRIFUGO_ALLOWED_ORIGINS`), and the image version in `neuralops/infra.env`.
4. Start the stack:
   ```bash
   docker compose -f docker-compose.neuralops.yaml \
     --env-file neuralops/infra.env --env-file neuralops/app.env --env-file neuralops/secrets.env \
     up -d
   ```
5. First run only: create the vector store, then set up the database:
   ```bash
   docker exec neuralops-postgres sh -c 'psql -U "$POSTGRES_USER" -c "CREATE DATABASE nexus_vectors"'
   docker exec neuralops-postgres sh -c 'psql -U "$POSTGRES_USER" -d nexus_vectors -c "CREATE EXTENSION vector"'
   for cmd in migrate seed_permissions seed_catalog; do
     docker exec -w /nexus/nucleus -e PYTHONPATH=/nexus/nucleus neuralops-nucleus \
       /opt/venv/nucleus-env/bin/python manage.py $cmd
   done
   ```
6. Claim the server from the app. For an air-gapped server, `manage.py create_owner` is the fallback.

Your reverse proxy must serve HTTPS (the web app is HTTPS, and browsers block it from talking to a plain-HTTP server) and pass websockets through to nginx on port 8095.

---

## Privacy: what leaves your server

- **Your data stays on your server:** projects, messages, files, the vector store, the audit log and your encrypted keys.
- **Model calls go to the provider you choose.** With a cloud model, prompts go to that provider under your own key. With Ollama, nothing leaves the machine. You can mix per persona.
- **Sign-in** goes through the NeuralOps portal by default. Every member can also set a local password, and then the server signs them in on its own, with no internet needed.
- **One anonymous usage total** is sent home (a count, no content). Nothing else.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| "This server is ARM" | ARM isn't supported yet. Use an x86_64 (Intel/AMD) server. |
| The address check fails (public-IP mode) | Open ports 80 and 443 in the cloud firewall / security group, then run the installer again. Or use Tailscale. |
| "An earlier NeuralOps database exists… but its secrets file is gone" | The old database can't be opened without its secrets. The installer backs it up and tells you how to clear it; then run it again. |
| Tailscale sign-in timed out | Run the installer again; it continues where it stopped. |
| The app can't reach the server | `neuralops status` and `neuralops address`; check the address opens in a browser. `neuralops logs nucleus` shows errors. |
| A reply fails with a model error | Re-check the key with **Check** in `/add-model`, and the provider's spending limit. |

---

## Uninstall

```bash
neuralops uninstall
```

It removes the containers and keeps your data. It then asks whether to delete **all** data too; that happens only if you type `DELETE`.

---

## Help and community

- Website and web app: [neuralopsnexus.ai](https://neuralopsnexus.ai)
- Book a demo: [cal.com/neuralopsnexus/30min](https://cal.com/neuralopsnexus/30min)
- Bugs and questions: [open an issue](https://github.com/noamanfaisal/neuralops-get/issues)
- Open-source MCP servers: [contextgraph](https://github.com/mapax-io/contextgraph) (your codebase as a graph over MCP, Apache-2.0) · csv-mcp (GB-scale CSVs over MCP, Apache-2.0)

---

NeuralOps Nexus is built by NeuralOps Nexus AI Inc., Edmonton, Canada. Free to self-host; compiled Docker image; source not yet public.
