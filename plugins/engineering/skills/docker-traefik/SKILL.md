---
name: docker-traefik
description: This skill should be used when a git worktree contains a docker-compose file and needs to be accessible via {worktree_name}.localhost using Traefik reverse proxy. It manages a shared Traefik instance and configures per-worktree routing automatically.
---

# Docker Traefik Worktree Router

This skill provides automatic HTTP routing to Docker Compose services running in git worktrees. Each worktree with a `docker-compose.yml` at its root gets a `{worktree_name}.localhost` URL via a shared Traefik reverse proxy.

## What This Skill Does

- **Manages a shared Traefik instance** that routes `*.localhost` to the correct worktree container
- **Auto-labels worktree services** so Traefik can discover and route to them
- **Starts/stops worktree services** with correct Traefik labels applied
- **Lists running worktree services** with their URLs

## Architecture

```
Browser
  │
  ├── feature-login.localhost ──► Traefik ──► feature-login container (port 80)
  ├── feature-signup.localhost ──► Traefik ──► feature-signup container (port 80)
  └── api-refactor.localhost   ──► Traefik ──► api-refactor container (port 80)
```

Traefik runs as a standalone container on the shared Docker network `traefik-net`. Each worktree's docker-compose services join this network and receive Traefik labels for routing.

## Prerequisites

- Docker and Docker Compose installed
- Worktrees managed via the `git-worktree` skill (in `.worktrees/` directory)
- Each worktree has a `docker-compose.yml` at its root
- The primary service in each compose file exposes an HTTP port

## How to Use

### Always Use the Manager Script

```bash
# Ensure Traefik is running (idempotent - safe to call multiple times)
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh up

# Start a worktree's services with Traefik routing
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh start <worktree-name> [service-port]

# Stop a worktree's services
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh stop <worktree-name>

# List all routed worktrees
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh list

# Tear down Traefik and the shared network
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh down
```

## Commands

### `up`

Ensures the shared Traefik reverse proxy is running. Creates the `traefik-net` Docker network if it doesn't exist and starts a Traefik container with the Docker provider enabled.

Traefik dashboard is available at `http://traefik.localhost`.

```bash
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh up
```

### `start <worktree-name> [service-port]`

Starts docker-compose services for the given worktree with Traefik labels injected. The worktree is expected to be at `.worktrees/<worktree-name>/`.

- `worktree-name` (required): Name of the worktree directory
- `service-port` (optional): The port the service listens on inside the container (default: `80`)

```bash
# Start with default port 80
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh start feature-login

# Start with custom port
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh start feature-login 3000
```

After starting, the service is accessible at `http://feature-login.localhost`.

### `stop <worktree-name>`

Stops and removes the docker-compose services for a worktree.

```bash
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh stop feature-login
```

### `list` or `ls`

Lists all worktrees that have a `docker-compose.yml` and shows whether their services are running, along with their routed URL.

```bash
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh list
```

### `down`

Stops Traefik and removes the shared network. Use when you're done with all worktree development.

```bash
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh down
```

## How It Works

### Shared Traefik Instance

The `up` command starts a Traefik container configured to:
1. Listen on port 80
2. Use the Docker provider to auto-discover services
3. Route based on `Host()` rules matching `{worktree-name}.localhost`

### Per-Worktree Routing

The `start` command:
1. Verifies the worktree exists and has a `docker-compose.yml`
2. Ensures Traefik is running (calls `up` if needed)
3. Generates a Traefik-aware docker-compose override file (`.docker-compose.traefik.yml`) that:
   - Adds Traefik labels to the first service
   - Connects the service to the `traefik-net` network
   - Sets the `Host()` rule to `{worktree-name}.localhost`
4. Runs `docker compose -f docker-compose.yml -f .docker-compose.traefik.yml up -d`

### DNS Resolution

`*.localhost` resolves to `127.0.0.1` on most systems by default (RFC 6761). No `/etc/hosts` editing required.

## Workflow Example

```bash
# 1. Create a worktree for a feature
bash ${CLAUDE_PLUGIN_ROOT}/skills/git-worktree/scripts/worktree-manager.sh create feature-login

# 2. Work on the feature, ensure docker-compose.yml exists
cd .worktrees/feature-login

# 3. Start it with Traefik routing
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh start feature-login 3000

# 4. Access at http://feature-login.localhost
# 5. Start another worktree
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh start feature-signup 3000

# 6. Both accessible simultaneously:
#    http://feature-login.localhost
#    http://feature-signup.localhost

# 7. See what's running
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh list

# 8. Clean up
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh stop feature-login
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh stop feature-signup
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh down
```

## Customization

### Changing the Default Service Port

If most of your services use a non-standard port (e.g., 3000), you can set the `TRAEFIK_DEFAULT_PORT` environment variable:

```bash
export TRAEFIK_DEFAULT_PORT=3000
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh start feature-login
```

### Multiple Services Per Worktree

If a worktree's docker-compose has multiple services and you want to route to a specific one, set `TRAEFIK_SERVICE`:

```bash
TRAEFIK_SERVICE=web bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh start feature-login 8080
```

## Troubleshooting

### "Cannot connect to feature-name.localhost"

1. Check Traefik is running: `docker ps | grep traefik`
2. Check the worktree service is running: `docker compose -f .worktrees/feature-name/docker-compose.yml ps`
3. Check Traefik dashboard at `http://traefik.localhost` for routing rules

### "Port 80 already in use"

Something else is using port 80. Stop it or change Traefik's port:

```bash
TRAEFIK_PORT=8080 bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh up
# Services then at http://feature-name.localhost:8080
```

### "Network traefik-net not found"

Run `up` again to recreate it:

```bash
bash ${CLAUDE_PLUGIN_ROOT}/skills/docker-traefik/scripts/traefik-manager.sh up
```
