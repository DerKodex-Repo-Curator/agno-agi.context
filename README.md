# @context — your work context, self-hosted

@context is a personal work-context proxy: a private CRM and knowledge base with an agent in front of it. Tell it things in plain language (*"met Kyle from Agno, wants a partnership, follow up next week"*) and it files contacts, notes, reminders, and specs for you — then briefs you back on demand (*"what's on today?"*). It plugs into Claude, Claude Code, Codex, ChatGPT, and Cursor over MCP, giving all your AI tools one shared source of truth about your work.

It's built for individuals who run their day across email, Slack, and a pile of AI tools — with privacy as a first principle: everything runs in your own infrastructure, and every byte lives in your own database.

> Built on [Agno](https://docs.agno.com)'s AgentOS runtime.

## How it works

| Piece | What it does |
|---|---|
| **One agent, role-gated tools** ([`agents/context.py`](agents/context.py)) | AgentOS verifies the caller on every request (JWT or Slack identity). The **owner** gets every tool; **guests** get capture-only — they can leave updates in your queue but never read or act. |
| **CRM** (`crm` Postgres schema) | Structured memory: contacts, projects, meetings, reminders, notes. No forms — @context maps what you say onto the right table. See [`docs/CRM.md`](docs/CRM.md). |
| **Knowledge base** (`knowledge/`, or a Git repo) | Specs, research notes, briefs. Filesystem-backed by default; back it with a private Git repo for production. See [`docs/KNOWLEDGE.md`](docs/KNOWLEDGE.md). |
| **Skills** ([`skills/`](skills/)) | Owner playbooks invoked in natural language: daily rundown, week plan, meeting prep, process-today, research, knowledge review. |
| **Workflows + schedules** ([`workflows/`](workflows/), [`app/schedules.py`](app/schedules.py)) | An hourly reminder sweep files due reminders into your queue; daily/weekly digests DM you on Slack when Slack is configured. |
| **External actions, double-gated** | Calendar changes pause for explicit approval (`requires_confirmation`); Gmail can *only* draft — it never sends. See [`docs/SECURITY.md`](docs/SECURITY.md). |

Guests (teammates and *their* agents) can @-mention your @context on Slack to drop updates in your queue — and your @context can do the same to theirs. That's the [context network](docs/NETWORK.md).

## Quick start

> Requires [Docker](https://www.docker.com/get-started/) installed and running.

```sh
git clone https://github.com/agno-agi/context.git
cd context

cp example.env .env
# In .env: set OPENAI_API_KEY, and set OWNER_ID to the email you sign in to os.agno.com with.

docker compose up -d --build
```

Confirm it's live at [http://localhost:8000/docs](http://localhost:8000/docs). Compose runs two containers: `context-api` (FastAPI + AgentOS, hot-reload in dev mode) and `context-db` (Postgres with pgvector).

## Interfaces

**AgentOS web UI.** Open [os.agno.com](https://os.agno.com), sign in with the email you set as `OWNER_ID`, click **Connect AgentOS → Local**, and enter `http://localhost:8000`. Chat with @context, view sessions, and approve actions from there.

**MCP (Claude Code, Codex, Claude Desktop, Cursor, ChatGPT).** @context ships AgentOS's built-in MCP server at `http://localhost:8000/mcp`, exposing one owner-only tool: `use_context`. Wire up every MCP client on your machine with one command:

```sh
python scripts/connect.py            # --dry-run to preview, --remove to undo
```

Once deployed, `./scripts/setup_context.sh` mints a self-issued JWT and points your clients at the live instance. Per-client details in [`docs/MCP.md`](docs/MCP.md).

**Slack.** The interface that lets your team (and their agents) talk to @context. Set `SLACK_BOT_TOKEN` and `SLACK_SIGNING_SECRET` and restart; setting the bot token also arms the scheduled daily/weekly digest DMs. Setup guide: [`docs/SLACK.md`](docs/SLACK.md).

## Deploy

@context runs anywhere that runs a Docker container. For Railway (the included path — requires the [Railway CLI](https://docs.railway.com/cli#installing-the-cli)):

```sh
cp .env .env.production   # deploy scripts read .env.production, falling back to .env
railway login
./scripts/railway/up.sh   # @context + Postgres on a private network, public domain for the API
```

Two things to know:

- **JWT is required at prod boot.** With `RUNTIME_ENV=prd`, AgentOS refuses to serve traffic without `JWT_VERIFICATION_KEY` — the safe default for an agent holding your inbox and CRM. `up.sh` pauses while you mint the key at [os.agno.com](https://os.agno.com) (**Connect AgentOS → Live** → enable **Token Based Authorization**, paste the public key into `.env.production`) or [bring your own JWT](https://docs.agno.com/agent-os/security/authorization/self-hosted).
- **Single-replica scheduler.** The deploy ships `numReplicas: 1` ([`railway.json`](railway.json)); the built-in scheduler assumes it. Before scaling out, read [`docs/SCALING.md`](docs/SCALING.md) and pin `INTERNAL_SERVICE_TOKEN`.

Afterwards: `railway logs --service agent-os` to watch it come up, `./scripts/railway/env-sync.sh` to push `.env.production` changes, `./scripts/railway/redeploy.sh` after code changes, and `./scripts/setup_context.sh` to wire your MCP clients to production. If Slack was set up locally, repoint its request URLs to the Railway domain ([`docs/SLACK.md`](docs/SLACK.md#moving-from-local-to-production)).

## Configuration

Copy [`example.env`](example.env) to `.env` — it documents every knob. The ones that matter most:

| Variable | Required | Description |
|---|---|---|
| `OPENAI_API_KEY` | yes | Models and embeddings. |
| `OWNER_ID` | prod | Comma-separated identities that count as the owner (JWT `sub` and/or Slack email). First is canonical. Unset means capture-only for everyone. |
| `OWNER_NAME` / `OWNER_TIMEZONE` | no | Display name; IANA timezone anchoring "today" and due-date math (default UTC). |
| `RUNTIME_ENV` | no | `dev` enables hot-reload and disables JWT (compose sets this); default `prd`. |
| `JWT_VERIFICATION_KEY` | prod | Public key from os.agno.com. AgentOS won't serve traffic in `prd` without it. |
| `SLACK_BOT_TOKEN` / `SLACK_SIGNING_SECRET` | no | Enable the Slack interface; the bot token alone activates the `slack` source and arms the digests. |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` / `GOOGLE_PROJECT_ID` | no | Gmail + Calendar; mint tokens with `python scripts/google_mint_tokens.py` ([`docs/GOOGLE.md`](docs/GOOGLE.md)). |
| `KNOWLEDGE_REPO_URL` / `KNOWLEDGE_GITHUB_TOKEN` | no | Back the knowledge base with a Git repo instead of local files. |
| `PARALLEL_API_KEY` | no | Authenticated web search (higher rate ceiling); recommended for production. |
| `DAILY_DIGEST_CRON` / `WEEKLY_DIGEST_CRON` | no | UTC crons for the Slack digests (defaults `0 13 * * *` / `0 22 * * 0`). |
| `DB_HOST` / `DB_PORT` / `DB_USER` / `DB_PASS` / `DB_DATABASE` | no | Postgres connection; defaults match compose. |

Never commit `.env` or `.env.production` — both are gitignored and carry real keys.

## Evals

The eval suite ([`evals/`](evals/)) regression-tests the security claim: *anyone can write, only you can read.* It needs a local venv:

```sh
./scripts/venv_setup.sh && source .venv/bin/activate

python -m evals                # full suite
python -m evals -v             # stream the agent runs
python -m evals --case <name>  # one case
```

Exit 0 on all-pass, non-zero otherwise.

## Source / links

- [`AGENTS.md`](AGENTS.md) — architecture and conventions (the source of truth for coding agents working on this repo).
- [`docs/`](docs/) — deep dives: [CRM](docs/CRM.md), [knowledge](docs/KNOWLEDGE.md), [MCP](docs/MCP.md), [Slack](docs/SLACK.md), [Google](docs/GOOGLE.md), [security](docs/SECURITY.md), [scaling](docs/SCALING.md), [the context network](docs/NETWORK.md).
- [Agno documentation](https://docs.agno.com) · [AgentOS introduction](https://docs.agno.com/agent-os/introduction) · [Agno on GitHub](https://github.com/agno-agi/agno) (drop a star if this is useful).
