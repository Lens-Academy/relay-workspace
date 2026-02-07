# Relay Workspace

Meta-repo for the Lens Relay ecosystem. Clone this first, then clone the child repos inside it.

## Architecture

```
                    ┌─────────────────────┐
                    │   Cloudflare R2     │
                    │ (lens-relay-storage) │
                    └────────┬────────────┘
                             │
Internet ── Cloudflare ── cloudflared ── relay-server (Rust, port 8080)
               Tunnel                        │
                                             │ webhooks
                                             ▼
                                       relay-git-sync
                                        │         │
                                        ▼         ▼
                                   lens-relay  lens-edu-relay
                                   (GitHub)    (GitHub)

Clients:
  - Obsidian + Relay.md plugin (real-time collaborative editing)
  - lens-editor (web-based editor, React + CodeMirror)
```

### Components

| Component | Repo | Description |
|-----------|------|-------------|
| **relay-server** | `Lens-Academy/lens-relay-server` | Fork of `No-Instructions/relay-server`. Rust-based CRDT sync server (y-sweet). Custom HMAC auth fixes for service accounts. |
| **lens-editor** | `Lens-Academy/lens-editor` | Web-based editor for relay documents. React + CodeMirror + yjs. Connects to relay-server via WebSocket. |
| **relay-git-sync** | `No-Instructions/relay-git-sync` | Syncs relay shared folders to GitHub repos via webhooks. Runs as Docker container on production server. |
| **Relay.md plugin** | `No-Instructions/Relay` | Obsidian plugin for real-time collaboration via relay-server. |

### Infrastructure

- **Relay server URL:** https://relay.lensacademy.org
- **Production server:** Hetzner VPS (46.224.127.155), Docker containers
- **Storage:** Cloudflare R2 bucket `lens-relay-storage`
- **Tunnel:** Cloudflare Tunnel (no inbound ports needed)
- **Relay ID:** `cb696037-0f72-4e93-8717-4e433129d789`

## Setup

### 1. Clone this workspace

```bash
git clone git@github.com:Lens-Academy/relay-workspace.git
cd relay-workspace
```

### 2. Clone child repos

```bash
# Relay server (our fork with HMAC auth fixes)
git clone git@github.com:Lens-Academy/lens-relay-server.git

# Web editor
git clone git@github.com:Lens-Academy/lens-editor.git

# Relay.md Obsidian plugin (upstream, read-only reference)
git clone https://github.com/No-Instructions/Relay.git relay-obsidian-plugin
```

If using jj:
```bash
jj git clone git@github.com:Lens-Academy/lens-relay-server.git
jj git clone git@github.com:Lens-Academy/lens-editor.git
jj git clone https://github.com/No-Instructions/Relay.git relay-obsidian-plugin
```

The Obsidian plugin source is important to have on hand — our relay-server and lens-editor need to stay compatible with the Relay.md client protocol (auth handshake, CRDT sync, file tokens, shared folder format).

### 3. Set up upstream remote for relay-server

```bash
cd lens-relay-server
git remote add upstream https://github.com/No-Instructions/relay-server.git
# or with jj:
jj git remote add upstream https://github.com/No-Instructions/relay-server.git
```

## Running relay-server

### With Docker (production-like)

On the production server, the relay-server runs as a Docker container:

```bash
docker build -t relay-server:custom .
docker run -d \
  --name relay-server \
  --restart unless-stopped \
  --network relay-network \
  --ulimit nofile=65536:524288 \
  -v /root/relay.toml:/app/relay.toml:ro \
  --env-file /root/auth.env \
  relay-server:custom
```

### With Cargo (local dev)

```bash
cd lens-relay-server
cargo build
cargo run -- serve relay.toml
```

Requires a `relay.toml` config file. See the relay-server README for config format.

## Running lens-editor

```bash
cd lens-editor
npm install
npm run dev
```

The editor connects to the relay server at the URL configured in its settings. During development, point it at `https://relay.lensacademy.org` or a local relay-server instance.

## Custom Relay Server Changes

Our fork (`lens-custom` branch) has two critical HMAC auth fixes on top of upstream:

1. **`auth.rs`** — `gen_doc_token_auto()`: Supports HMAC keys in document token generation
2. **`server.rs`** — `gen_file_token_auto()`: Generates file tokens for server/prefix tokens in download-url

**Why:** Relay.md doesn't provide service account API keys. Without these fixes, you can have either image sync between users OR git-sync, but not both. Our fix enables both simultaneously.

Additional work on `lens-custom`:
- Link indexer for wikilink extraction and backlink tracking
- Folder-content mapping for multi-folder support

## Git Sync

Two shared folders are synced to GitHub:

| Obsidian Folder | GitHub Repo | Branch |
|-----------------|-------------|--------|
| Lens | [Lens-Academy/lens-relay](https://github.com/Lens-Academy/lens-relay) | main |
| Lens Edu | [Lens-Academy/lens-edu-relay](https://github.com/Lens-Academy/lens-edu-relay) | staging |

## Known Issues

- **WebSocket FD leak** in relay-server (sockets accumulate in CLOSE-WAIT). Workaround: `--ulimit nofile=65536:524288` extends time-to-restart to ~39 days.
