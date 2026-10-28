# VPS setup — cz-agents (@czagents) + OfficeCLI in the gateway

Deploy the Czech/EU company-diligence MCP servers and OfficeCLI into the
`hex-ai-mcp` gateway on the VPS. Everything is **self-hosted** (no third-party
API keys): the `@czagents/*` servers call the public Czech/EU government
registries directly, and OfficeCLI is a local binary.

**Target:** VPS host `HEX-AI`, user `hex-chat`, deploy checkout `~/hex-ai-mcp`.
**Note:** home dir `~` = `/home/hex-chat`, so `~/officecli` = `/home/hex-chat/officecli`.
(this repo, branch `node24`). LibreChat runs as docker on the same host.

---

## Phase 0 — pull `node24` and bring the infra networks up

The compose references the `librechat_default` **external** docker network, so
it must exist before `docker compose up`:

```bash
cd ~/hex-ai-mcp
git fetch origin
git checkout node24
git pull origin node24

# Create the external LibreChat network if LibreChat hasn't already.
# Idempotent check:
docker network ls | grep librechat_default || docker network create librechat_default

# .env should already have (edit if missing):
#   MCP_PROXY_URL_ALLOWED_HOSTS=google-mcp     # lets the gateway dial google-mcp by name
POSTGRES_PASSWORD="$POSTGRES_PASSWORD" BETTER_AUTH_SECRET="$BETTER_AUTH_SECRET" \
  docker compose up -d --build
```

> `docker compose config` exits 0 on this tree, so the mounts/network are valid.
> The new mounts baked in by the compose change:
> - `~/officecli` → **`/home/officecli`** (rw) — OfficeCLI binary + pass-through files
> - `~/czagents` → **`/home/czagents`** (rw) — cz-agents data (SANCTIONS_DB mirror)
> - `npm_cache` → `/home/nextjs/.npm` — persists `npx` package installs across container recreates

---

## Phase 1 — OfficeCLI

OfficeCLI ships as a **single self-contained .NET binary** with a built-in MCP
stdio server (`officecli mcp`). No wrapper needed.

### 1. Install the linux-x64 binary into the pass-through dir

```bash
mkdir -p ~/officecli && cd ~/officecli
curl -fSL -o officecli https://github.com/iOfficeAI/OfficeCLI/releases/latest/download/officecli-linux-x64
chmod +x officecli
./officecli --version          # e.g. 0.x
./officecli mcp --help         # confirm the built-in MCP subcommand exists
```

This dir is bound into the gateway at `/home/officecli`. Use the same dir as the
**pass-through workspace** where Drive-downloaded `.docx/.xlsx/.pptx` files are
edited (LibreChat and the `officecli` STDIO process both reach it by absolute
path).

### 2. Register the STDIO server (gateway UI)

Server → Add → **STDIO**:

| field | value |
|---|---|
| name | `officecli` |
| type | `STDIO` |
| command | `/home/officecli/officecli` |
| args | `["mcp"]` |

- Read/edit/create covers `.docx/.xlsx/.pptx` (comments, highlight, tracked
  changes, chart/pivot editing, merge/template, dump/batch).
- `officecli view screenshot` needs a headless browser; the core read/edit tools
  do not.

---

## Phase 2 — @czagents (self-hosted, npm)

The gateway image ships **node 24 + npm**, so each package runs via `npx`; the
`npm_cache` volume keeps installs warm across restarts. Register one STDIO
server **per package** (the gateway aggregates them under a namespace).

### 2.1 Register the servers (UI: Server → Add → STDIO)

| name | command | args | env (JSON) |
|---|---|---|---|
| `cz-ares` | `npx` | `["-y","@czagents/ares"]` | `{}` |
| `cz-eu-registry` | `npx` | `["-y","@czagents/eu-registry"]` | `{}` |
| `cz-cnb` | `npx` | `["-y","@czagents/cnb"]` | `{}` |
| `cz-isir` | `npx` | `["-y","@czagents/isir"]` | `{"ISIR_SOAP_ENABLED":"1"}` |
| `cz-adis` | `npx` | `["-y","@czagents/adis"]` | `{"ADIS_SOAP_ENABLED":"1"}` |
| `cz-dd` | `npx` | `["-y","@czagents/dd"]` | `{"SANCTIONS_DB":"/home/czagents/sanctions.db","ADIS_SOAP_ENABLED":"1"}` |

Notes:
- `@czagents/ares` is the ARES business-register server (superset of the
  vzeman/ares-mcp-server, which we intentionally did **not** deploy — this one
  already covers the EU-country roadmap via `@czagents/eu-registry`).
- `@czagents/dd` = due-diligence aggregator (ARES + sanctions + ISIR + ADIS +
  statutory-chain/UBO). It maintains a **local SANCTIONS_DB** mirror — create
  the dir `~/czagents` (mounted at `/home/czagents`) and let the package
  build the DB on first use.
- Some `dd`/`payqr` tools are tier-gated in the package; core lookup is free.
- Optional extra packages you can add the same way: `@czagents/realestate`,
  `@czagents/payqr`.

### 2.2 First-run verification of the local DB path

The gateway spawns with `cwd` **inherited from the gateway process** (the STDIO
`cwd` field is not persisted per server), so point any file-backed env at an
**absolute** path under a mounted workspace — e.g. `/home/czagents/...` above,
not `~/...`.

---

## Phase 3 — Namespace & expose-all

1. Gateway UI → Namespaces → create (or reuse) a `cz` namespace.
2. Add all `cz-*` servers + `officecli` to it (Admin → Namespace → Servers).
3. Default = **expose all tools** on that namespace's endpoint.
4. Restrict later via the per-server/per-namespace tool `INACTIVE` toggles (and
   LibreChat per-agent MCP toggles) — deliberately left open for now.

---

## Phase 4 — Verify

- **Per-server connect:** gateway Servers list shows each as connected/active and
  the Tools table is populated (e.g. `cz-ares` = lookup_by_ico, search_companies,
  get_statutaries, validate_dic, check_vat_payer, …; `cz-eu-registry` =
  search_company / get_company / lookup_company_by_vat).
- **ARES:** in LibreChat ask e.g. *"ověř firmu s IČO 45274649"* → `cz-ares`.
- **EU:** `cz-eu-registry` search across the 16 countries.
- **OfficeCLI:** in LibreChat, Drive-download a `.docx` to `/home/officecli`,
  then have the agent edit it with `officecli` and dump/save back.

---

## Security notes

- **Self-hosted & keyless:** `@czagents/*` call public government registries;
  no outbound API key, no hosted `*.cz-agents.dev` dependency.
- **Expose-all is the temporary default** (per operator). The gateway's
  per-namespace tool `INACTIVE` toggles and LibreChat per-agent toggles are the
  restriction mechanism once you decide.