# Build your own always-on agent box (herdr in the cloud)

Paste everything below the line into Claude Code, Codex, or any coding agent that can run shell
commands and connect MCP servers. It sets up a small cloud server where your coding agents keep
working after you close your laptop. You reach it from a browser tab or your phone, and it sits
behind a login so nobody else can.

You need a DigitalOcean account, a domain you control that's on Cloudflare (the free plan is
fine), and about an hour. The server is a monthly bill, from about $24 a month for a small one;
the agent shows you the exact price and waits for a yes before it creates anything.

---

You are setting up a personal, always-on agent workstation in the cloud for me. Work through the
phases in order. Check each one before you start the next, and tell me in one or two lines what
you did and what you checked.

## What we're building

- A DigitalOcean Droplet running Ubuntu 24.04, with a separate block storage Volume mounted at
  `/data` so my work survives a rebuild.
- Docker on the Droplet, running one container that holds the whole toolset:
  - [herdr](https://herdr.dev/), a terminal workspace manager for coding agents, running as a
    headless server so sessions stay alive with nobody attached
  - Claude Code and the Codex CLI
  - [ttyd](https://github.com/tsl0922/ttyd), so herdr opens in a browser tab
  - [code-server](https://github.com/coder/code-server), VS Code in the browser
  - [dufs](https://github.com/sigoden/dufs), a small file browser for uploads and downloads
  - sshd on port 2222, key-only, for terminal apps
  - [supervisord](https://supervisord.org/) keeping all of the above running
- A [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
  running on the host, so the server has no open inbound ports at all.
- [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/access-controls/) in front
  of every hostname, so only my email address can sign in (one-time PIN to my inbox).

## Before you touch anything, ask me for

1. The domain to use (it must already be on my Cloudflare account) and the subdomains I want.
   Suggest `agent.`, `code.`, `files.` and `ssh.` as defaults.
2. The email address or addresses allowed through Cloudflare Access.
3. The Droplet region and size. Recommend 4 vCPU / 8 GB as the floor for running several agents
   at once, and show me the monthly price before creating it.
4. My SSH public key. If I don't have one, walk me through `ssh-keygen -t ed25519`.
5. Which agents to install: Claude Code, Codex, or both.

Never ask me to paste an API token, password or private key into this chat. Have me store
tokens in my OS keychain or an environment variable you read from, and never print them.

## Connect these MCP servers first

- **DigitalOcean**: the hosted Droplets server at `https://droplets.mcp.digitalocean.com/mcp`.
  Docs: https://docs.digitalocean.com/reference/mcp/ and the setup page at
  https://docs.digitalocean.com/reference/mcp/configure-mcp/. The tool list is at
  https://docs.digitalocean.com/reference/mcp/droplet-mcp-tools/. There is also a local server,
  `npx -y @digitalocean/mcp` (https://github.com/digitalocean-labs/mcp-digitalocean). The
  networking server covers firewalls; the full list is at
  https://docs.digitalocean.com/reference/mcp/mcp-tools/.
- **Cloudflare**: `https://mcp.cloudflare.com/mcp` covers the whole Cloudflare API, and
  `https://docs.mcp.cloudflare.com/mcp` searches Cloudflare's docs. Overview:
  https://developers.cloudflare.com/agents/model-context-protocol/cloudflare/servers-for-cloudflare/

If a server needs OAuth, start the flow and tell me what to click. If an MCP can't do something,
fall back to the provider's API or CLI (`doctl`, `cloudflared`) and say so.

## Phase 1: provision

1. Create a DigitalOcean
   [Cloud Firewall](https://docs.digitalocean.com/products/networking/firewalls/) that allows
   inbound TCP 22 from my current public IP only. Nothing else inbound.
2. Create a [Volume](https://docs.digitalocean.com/products/volumes/) (start at 50 GB) in the same
   region.
3. Create the Droplet with the firewall and Volume attached, my SSH key, and the cloud-init file
   from Phase 2.
4. Mount the Volume at `/data` through `/etc/fstab` using its UUID with `nofail`, so a missing
   Volume can't stop the server booting.

## Phase 2: harden the host with cloud-init

Write a [cloud-init](https://cloudinit.readthedocs.io/) file that does all of this on first boot:

- Creates a non-root user `agent` with sudo and my SSH key, and removes the default `ubuntu` user.
- Adds an sshd drop-in that turns off password login and root login.
- Turns on `unattended-upgrades` for security patches, plus `fail2ban`, `chrony` and `earlyoom`.
- Adds a swap file and sets `vm.swappiness=20` and `fs.inotify.max_user_watches=524288` (agents
  and editors watch a lot of files).
- Installs Docker from Docker's own apt repo (https://docs.docker.com/engine/install/ubuntu/).

## Phase 3: the container

Write a Dockerfile on `ubuntu:24.04` that installs:

- Node 22 (NodeSource), git, the GitHub CLI, `uv`, mosh, tmux, ripgrep, jq, and a headless Chrome
  if I want browser automation.
- herdr through its installer (https://herdr.dev/docs/install/), then
  `herdr integration install claude` so Claude Code shows up as an agent in herdr. Read
  https://herdr.dev/docs/agents/ for the other agents.
- Claude Code (https://docs.anthropic.com/en/docs/claude-code/setup) and the Codex CLI
  (https://github.com/openai/codex).
- ttyd, code-server and dufs from their release pages.
- A non-root user `agent` with UID 1000. Everything runs as that user.

Put every piece of state under `/data`: the agent's home directory, herdr's session state
(https://herdr.dev/docs/session-state/), the agents' login files and my workspaces. Rebuilding the
container must not log me out or lose a session.

Write a `docker-compose.yml` that mounts `/data:/data`, sets `restart: always`, and loads secrets
from a root-only env file at `/data/.box-env` (mode 600). Secrets never go in the compose file.

### Networking: get this right the first time

- Use `network_mode: host`. ttyd, code-server and dufs should bind `127.0.0.1` only. If you use
  `ports:` instead, Docker forwards to the container's ethernet address, never its loopback, so
  every web service gives a connection reset while sshd (bound to 0.0.0.0) looks fine.
- If you ever do publish a port, bind it to `127.0.0.1:` explicitly. Docker writes its own
  iptables rules, and a bare published port gets past ufw.
- cloudflared runs on the host as a systemd service, not inside the container, and reaches every
  service over localhost.

### herdr has to be a session leader

Run herdr's server through a small wrapper so it outlives every SSH or browser connection:

```bash
#!/usr/bin/env bash
set -uo pipefail
cleanup() { /usr/local/bin/herdr server stop >/dev/null 2>&1 || true; }
trap 'cleanup; exit 0' TERM INT
setsid --wait /usr/local/bin/herdr server &
wait $!
```

Without `setsid`, herdr reports that it isn't a detached server, and `herdr --remote` asks
"restart the remote server now?" on every attach. `--wait` keeps supervisord able to restart it
after a crash. Details: https://herdr.dev/docs/persistence-remote/

### supervisord gotchas

- herdr refuses to start without a real terminal. ttyd provides one, so point ttyd at a script
  that attaches to herdr.
- Don't pass `--client-option key=value` to ttyd in a supervisord `command=`. supervisord splits on
  `=`, ttyd takes the value as its command, and every WebSocket dies with an "origin service EOF"
  error that looks like a tunnel fault.
- Give code-server and dufs a password from the env file too, as a second lock behind Access.

## Phase 4: Cloudflare Tunnel and Access

1. Create a named tunnel
   (https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/),
   install `cloudflared` on the host, and run it as a systemd service with its token in a root-only
   file.
2. Route the hostnames: the browser terminal to ttyd, `code.` to code-server, `files.` to dufs,
   and `ssh.` to `ssh://localhost:2222`.
3. Create an Access application for each hostname with one policy: allow my email addresses
   through one-time PIN. Set the session length to something I can live with on a phone (a week
   is reasonable).
4. For SSH from a laptop, set up `cloudflared access ssh` as a ProxyCommand
   (https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/use-cases/ssh/) and write the
   `~/.ssh/config` block for me.
5. Optional, for phone SSH apps (they can't run a ProxyCommand): a private network route plus the
   Cloudflare One / WARP client
   (https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/private-net/
   and https://developers.cloudflare.com/cloudflare-one/team-and-resources/devices/warp/). Mosh
   (https://mosh.org/) is worth adding for flaky mobile connections. The browser terminal works on
   a phone with no app at all, so mention that as the zero-setup option.

If a new hostname fails TLS right after the zone activates, wait a few minutes for the edge
certificate before you debug anything else.

## Phase 5: sign in the agents

Open the browser terminal and have me run `claude` and `codex` once each, so I can do the login
flow myself. The login files land under `/data`, so this is the only time I do it. Then set
`gh auth login` the same way if I want GitHub access.

## Phase 6: verify, then lock down

Check every one of these, and show me the result:

- Each hostname answers with a redirect to Cloudflare Access in an incognito window (HTTP 302),
  and loads after I sign in.
- herdr survives a container restart with my session still there:
  `docker compose restart`, then reattach.
- Rebooting the Droplet brings everything back with no manual steps.
- SSH works through the tunnel.

Only after all of that passes, lock down:

1. Bind the host's sshd to `127.0.0.1` so the tunnel is the only way in.
2. Remove the port 22 rule from the Cloud Firewall. The server now has no public inbound ports.
3. Confirm port 22 no longer answers on the public IP.

Before step 2, tell me that if the tunnel ever breaks, the way back in is DigitalOcean's Recovery
Console (https://docs.digitalocean.com/products/droplets/how-to/recovery/recovery-console/). Ask
me to confirm before you do it.

## Phase 7: hand-off

Give me a one-page summary: every URL, how to SSH in, what lives on `/data`, how to rebuild the
container, how to check logs (`docker compose logs`, `supervisorctl status`), the monthly cost,
and anything you skipped or couldn't verify. Save it as `BOX.md` in my home directory on the
server, with no secrets in it.

## Rules for the whole job

- Show me the price before creating anything that costs money, and wait for a yes.
- Confirm before deleting, overwriting or locking anything down.
- Never print, log or commit a token, password or private key.
- If a step fails twice, stop, tell me what you tried and what you saw, and ask how to proceed.
- Reference docs: herdr (https://herdr.dev/docs/quick-start/,
  https://herdr.dev/docs/configuration/, https://herdr.dev/docs/socket-api/,
  https://github.com/herdrdev/herdr), the MCP spec (https://modelcontextprotocol.io/), and
  Cloudflare's MCP guide (https://developers.cloudflare.com/agents/model-context-protocol/).
