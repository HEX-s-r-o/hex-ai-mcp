# NEXT SESSION BRIEFING — HEX AI Chat: PDF tools + Drive office-file editing

Handoff prepared: 2026-10-19. Use this as the opening context for a fresh research
conversation. It captures everything from the previous session (the MetaMCP / Google
MCP / LibreChat integration) plus the two new feature requests.

---

## 1. The two NEW requests (research yardstick for this session)

### R1 — PDF tools inside HEX AI Chat
Users of the LibreChat fork need PDF capabilities directly in chat:
- read a PDF,
- add inline **comments**,
- **highlight**,
- mark / annotate (draw, strike, sticky notes),
- **add text** onto the page.

Open questions to research (do NOT assume we already know):
- Is there a maintained **MCP server** that exposes PDF read + comment/highlight/annotate/text
  tools? If yes: OAuth story, file-store model, STDIO vs HTTP worker, how files are passed
  in/out (paths vs inline bytes), multi-user safety when editing shared PDFs.
- Or should this be LibreChat-native: File Context / upload-as-text already extract PDF text
  (OCR via Mistral is configured in `librechat.yaml`, see `ocr:` section); highlight/comment
  needs a real annotation engine, so an embedded editor or an attached MCP server is likelier
  than pure text extraction.
- Where edited PDFs land: LibreChat attachments, Drive, or a managed file store; how the
  gateway namespace / tool restriction would gate which users/agents get which PDF tools.

### R2 — Drive office-file download + edit inside HEX AI Chat
- Download Google Drive files: **.docx / .pptx / .xlsx** (and probably .pdf, .txt).
- Edit them **inside HEX AI Chat** and save back.

Current partial coverage (already built in the previous session — verify against it):
- Drive download exists in BOTH Google servers already: a-bonus fork `downloadFile` and
  quinnjr `drive_download_file` (Drive supports export mime types: PDF/DOCX/CSV/etc.).
- What does NOT exist yet: an in-chat **office editor** (LibreChat has no native docx/pptx/
  xlsx editor; it has Code Interpreter + File Context + Artifacts as candidates).
- Research: a docx/xlsx/pptx editing MCP server? Round-tripping via LibreChat file upload +
  Code Interpreter? Save-back path (Drive update requires `drive_update_file`/API call with
  the edited bytes)?

Decisions wanted at the end of R1+R2 research: **one combined "office + PDF" MCP server vs
two separate servers**, which gateway namespace/endpoint they expose under, per-agent tool
restriction, and whether LibreChat-native features (OCR/File Context/Code Interpreter) cover
the "read" side so we only need MCP for the "write/annotate" side.

---

## 2. Stack and topology (as it stands)

```
Users → HEX AI Chat (LibreChat fork, hex-ai-chat / hex-ai-chat-private)
          └─ MCP endpoint (one gateway endpoint, key-authenticated)
              └─ MetaMCP gateway (hex-ai-mcp) on VPS, docker
                    ├─ server "google"      = OUR FORK of a-bonus/google-docs-mcp
                    │   (docs, drive, sheets, gmail, calendar, utils, script, slides)
                    └─ server "google-mcp"   = quinnjr/google-mcp (STREAMABLE_HTTP :3015)
                        (docs, sheets, drive, gmail, calendar, slides, tasks, contacts,
                         youtube, forms, chat, meet)
```

- Gateway host: VPS (`HEX-AI`, user `hex-chat`), docker compose in `~/hex-ai-mcp`.
- Private LibreChat config: `hex-ai-chat-private/` (`librechat.yaml`, `.env`), `dev.sh` runs
  the fork `hex-ai-chat` locally on the dev machine; on the VPS LibreChat runs as docker.
- Dev box (Arch, `LuicipherOS`/`work`): podman for local gateway test runs (port 12008).

## 3. Google credential model (the part that caused the most friction — read this!)

**One OAuth consent per server, minted with the FULL scope that server needs.** A refresh
token is bound to the scopes granted AT CONSENT — adding a tool later means re-consent.

- **a-bonus fork token**: `XDG_CONFIG_HOME=/data/google-docs-mcp/token.json`
  (the `abonus-data:/data` bind mount in compose). Format: `{ "refresh_token": ... }` only.
  Scopes: docs, drive, spreadsheets, script.external_request, script.projects, gmail.modify,
  calendar.events, **presentations** (added this session).
- **quinnjr token**: `/root/.local/share/google-mcp/tokens.json` inside the `google-mcp`
  container, VPS host sources `~/google-mcp/config/credentials.json` +
  `~/google-mcp/data/tokens.json` (bind-mounted). Format: FULL `Auth.Credentials`
  (access_token, refresh_token, expiry_date, token_type, scope) — different from a-bonus!
  Scopes: documents, spreadsheets, drive, tasks, calendar, gmail.modify+send, contacts,
  presentations, youtube, forms.body+responses, chat.*, meetings.*
- **Auth flow for the VPS problem:** the OAuth callback is `http://localhost:<port>`, which a
  desktop browser can never reach back to the VPS → mint the token on the DESKTOP (where the
  browser works) with the same client, then `scp` the produced `tokens.json` into the VPS
  volume. Never run `google_auth` targeting the VPS container's localhost.
- **Same client/account, different tokens is fine:** Google allows multiple refresh tokens
  per client+user; granting quinnjr more scopes does not invalidate the fork token. They do
  die together if the whole app is revoked at myaccount.google.com/permissions.

## 4. What is DONE (verified working)

Gateway repo `hex-ai-mcp` — branch `fix/dedupe-and-admin`, all pushed to GitHub
(`HEX-s-r-o/hex-ai-mcp.git`), working tree clean:
- `bulkUpsert` tool dedupe (fix for `UNIQUE(name, mcp_server)` crashes).
- Bootstrap admin (`BOOTSTRAP_USERS` with `"role":"admin"` → auto admin, idempotent).
- Compose now builds local image (`build:`) instead of the old GHCR image; volumes:
  branding logo (ro), `abonus-data:/data`, `uv_cache`, `npm_cache`.
- HEX branding: default logo `/branding/logo.png`, favicon via `app/icon.png` (HEXLOGO.png).
- `.gitignore` hardened (`abonus-data/`, `.playwright-mcp/`, keys).

Fork `google-docs-mcp` (a-bonus + slides) — branch `feat/slides`, local only, NOT pushed yet:
- New group `src/tools/slides/` (9 tools incl. createPresentation/getPresentationInfo/addSlide/
  duplicateSlide/deleteSlide/insertTextbox/replaceAllText/insertImage/exportPresentationPdf).
- Two fixes: early-return `googleSlides` client bug; `driveFolderId` invalid on Slides create →
  create-in-root then Drive `files.update({addParents})`.
- Build/test green (tsc + 395 vitest). Deployed to VPS, `createPresentation` verified working,
  folder placement verified.
- `slides` added to `MCP_TOOL_GROUPS` default; `auth.ts` SCOPES now includes `presentations`.

quinnjr `google-mcp`:
- Container built+authenticated on VPS, running Streamable HTTP on 0.0.0.0:3015.
- credentials.json + tokens.json placed in `~/google-mcp/config` and `~/google-mcp/data`,
  bind-mounted into the container.

## 5. What is IN FLIGHT / HAS OPEN LOOSE ENDS

- **quinnjr connect in MetaMCP was REFUSED by the URL guard**:
  `MCP proxy remote URL refused: "google-mcp" resolves into private (RFC 1918)`. Fix is the
  escape hatch env `MCP_PROXY_URL_ALLOWED_HOSTS=google-mcp` (comma separated; add to VPS
  gateway `.env`, then `docker compose up -d` and reconnect). **Verify this actually landed**
  — it was prescribed, and the last confirmed state was still the refusal error.
- **Namespace join**: add `google-mcp` to the same namespace as `google` so the one endpoint
  aggregates both tool sets. Registration: Type `STREAMABLE_HTTP`, URL
  `http://google-mcp:3015/mcp` — NO bearer token, NO custom headers (quinnjr does its own OAuth
  internally; the LibreChat-facing key auth stays on the gateway endpoint).
- **LibreChat SSRF allowlist**: for local dev, `mcpSettings.allowedAddresses` needed
  `localhost:12008` etc.; for VPS docker, the LibreChat container must reach the gateway over
  its network (container-name DNS), not `localhost`. Adjust accordingly for any NEW servers.
- **Tool restriction** (security-relevant once office/PDF tools land): MetaMCP per-namespace
  tool status `INACTIVE` toggles (gateway dashboard → Namespace → Tools table) plus
  `MCP_TOOL_GROUPS` (whole groups) on STDIO servers; LibreChat per-agent MCP tool toggles.
  quinnjr exposes WRITES for tasks/chat/meet/forms/youtube by default — decide what's exposed.
- **`fix/dedupe-and-admin` PR** against `umbrella` is still open/pending (gateway-side changes
  unmerged upstream).
- **`feat/slides` fork not pushed** — needs `HEX-s-r-o/google-docs-mcp` repo to exist first.
- Fork `npm audit`: ~20+ inherited vulnerabilities from a-bonus dep tree (dev tooling mostly);
  `npm run format:check` (prettier) must pass if contributed upstream.

## 6. Key files / dirs referenced in this session

- `hex-ai-mcp/` … gateway (branch fix/dedupe-and-admin; PR pending)
- `hex-ai-mcp/.agent/CONTINUITY.md` … running agent log (full history incl. earlier sessions)
- `~/google-docs-mcp/` … a-bonus fork (feat/slides; NOT backed up remotely yet)
- `~/google-mcp/` … quinnjr source; `config/credentials.json` + `data/tokens.json` (VPS)
- `hex-ai-chat-private/librechat.yaml` … LibreChat servers incl. `google-read_only`
  (`requiresOAuth: false`, static key header) and (present) gmail/googledrive/googlecalendar/
  googledocs/googleslides/googlesheets official-Google entries marked `requiresOAuth: true`
  which were deliberately NOT authenticated.
- `~/hex-ai-mcp/.env` … gateway secrets: BETTER_AUTH_SECRET, POSTGRES_PASSWORD,
  BOOTSTRAP_USERS, OAUTH TTLs, MAX_*_CONNECTIONS, GOOGLE_CLIENT_ID/SECRET.

## 7. Rules and constraints carried into this session

- No credentials in code/comments/commits. Commit author is `lui-do` for HEX repos.
- `feat/slides` and gateway branch work stays out of `main`/`umbrella` until PR-approved
  (gateway) / pushed to a HEX fork (google-docs-mcp).
- `example.env` is the annotated env reference; compose refuses to start without
  `POSTGRES_PASSWORD` and `BETTER_AUTH_SECRET` (`:?`).
- LibreChat fork docs live at `hex-ai-chat/docs/*` (e.g. mcp-apps.md) — consult before wiring
  new MCP servers into it.
- Do not silently run `npm audit fix` on the a-bonus fork; review the lockfile strategy
  separately.
