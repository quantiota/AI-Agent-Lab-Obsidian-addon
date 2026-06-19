# AI Agent Lab — Obsidian add-on

An optional **Obsidian** knowledge base for the [AI Agent Lab](https://github.com/quantiota/AI-Agent-Lab),
served at `obsidian.<your-domain>`. It is **not shipped with the lab core** — drop these
files into a running lab and bring it up with a compose overlay. Nothing in the core
(`docker-compose.yaml`, nginx `default.conf.template`) is overwritten.

Obsidian is the real desktop app (`lscr.io/linuxserver/obsidian`) streamed to the
browser over KasmVNC, gated by the lab's **Authelia SSO** (same login as vscode/grafana —
no extra password). The vault is the lab's existing `notebooks` folder, so **the Claude
agent in code-server reads and writes the same notes** — linked Markdown notes and AI
working side by side.

## AI Agent Lab + Obsidian = think, link, and let the agent work your notes

- Keep research notes, literature, and a lab journal as plain Markdown
- Link ideas with `[[wiki-links]]` and the graph view
- The Claude agent (in code-server) edits the same vault directly — summarise papers,
  generate linked notes, cross-reference experiments

## What's in here

```
docker/
  docker-compose.obsidian.yml         → compose overlay (obsidian service)            (copy to docker/)
  nginx/conf.d/obsidian.conf.template → obsidian vhost (own :80 + :443, Authelia SSO) (copy to docker/nginx/conf.d/)
```
No source folder to build (Obsidian ships as a prebuilt image), and no extra env vars
(access is via Authelia, which is already configured in the lab).

## Prerequisites

- A running AI Agent Lab (`docker/` with `docker-compose.yaml`, nginx, Authelia, and `.env` holding `DOMAIN`).
- A **DNS A record** `obsidian.<your-domain>` → the lab's server IP.
- TLS for `obsidian.<your-domain>` (see step 2).

## Install

1. **Copy the files into the lab's `docker/` tree:**
   ```bash
   cp -r AI-Agent-Lab-Obsidian-Addon/docker/* /path/to/AI-Agent-Lab/docker/
   ```

2. **TLS** — obtain a cert for `obsidian.<your-domain>` (the lab already runs certbot). Example http-01:
   ```bash
   cd /path/to/AI-Agent-Lab/docker
   docker compose run --rm certbot certonly --webroot -w /usr/share/nginx/html -d obsidian.$DOMAIN
   ```

3. **Authorise the subdomain in Authelia.** Add `obsidian.<your-domain>` to the
   `one_factor` rule's `domain:` list in `authelia/configuration.yml` (alongside
   `vscode`, `questdb`, `aiagentui`, `grafana`), then restart Authelia:
   ```bash
   docker compose restart authelia
   ```
   Without this, Authelia's `default_policy: deny` blocks the subdomain.

4. **Start with the overlay** (always pass BOTH `-f` files):
   ```bash
   cd /path/to/AI-Agent-Lab/docker
   docker compose -f docker-compose.yaml -f docker-compose.obsidian.yml up -d
   ```

5. **Open the vault.** Visit `https://obsidian.<your-domain>`, sign in with the lab's
   normal Authelia login, then in Obsidian choose **Open folder as vault** → `/config/vault`.
   That folder is the lab's `notebooks/`, so code-server sees the same files at
   `/home/coder/project` and the Claude agent can edit them.

## Day-to-day

Always include both compose files for any command touching Obsidian:
```bash
docker compose -f docker-compose.yaml -f docker-compose.obsidian.yml <cmd>
```

## Uninstall

```bash
cd /path/to/AI-Agent-Lab/docker
docker compose -f docker-compose.yaml -f docker-compose.obsidian.yml down
rm -f docker-compose.obsidian.yml nginx/conf.d/obsidian.conf.template nginx/conf.d/obsidian.conf
# (optional) remove the obsidian-config volume and the obsidian rule from authelia/configuration.yml
```

## Notes / caveats

- **Overlay model** — the core `docker-compose.yaml` and nginx `default.conf.template` are
  never modified, so the add-on survives lab updates. The only "shared" touch is dropping
  `obsidian.conf.template` into `nginx/conf.d/` (auto-loaded by `include conf.d/*.conf`).
- **Auth = Authelia SSO** — the vhost includes the lab's shared `authelia-location.conf`
  + `authelia-authrequest.conf` snippets, so Obsidian uses the same single sign-on as the
  rest of the lab (no separate password). KasmVNC itself runs without its own gate, behind Authelia.
- **Authelia rule** — the `obsidian.<DOMAIN>` rule lives in `authelia/configuration.yml`.
  If you ever regenerate that file from its `.sample`, re-add the rule (step 3) — or add it
  to your `.sample` to make it persistent.
- **Shared vault** — the vault is the lab's existing `../notebooks` folder (mounted at
  `/config/vault` in Obsidian, already at `/home/coder/project` in code-server). No new mount
  is added to the core `vscode` service. Keep `PUID`/`PGID` matching the code-server user (1000).
- **KasmVNC** streams a desktop over WebSocket; the vhost sets the `Upgrade`/`Connection`
  headers and a long `proxy_read_timeout` so the session stays alive.
