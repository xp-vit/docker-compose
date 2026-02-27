# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Docker Swarm deployment orchestration for multiple microservices across 3 EC2 nodes. Uses Traefik as reverse proxy with automatic SSL, and GitHub Actions for CI/CD. Stack name: `patotski`.

## Services

- **traefik** - Reverse proxy with Let's Encrypt SSL (ports 80, 443)
- **zaqlick-app** - Main application at `app.zaqlick.com` (Spring Boot)
- **zaqlick-social-data-parser** - Data parser at `data.patotski.com` (Spring Boot)

## Common Commands

```bash
# Deploy/update the stack (idempotent)
docker stack deploy --with-registry-auth -c docker-compose.yml patotski

# Force update a specific service
docker service update --force patotski_zaqlick-app

# View logs
docker service logs -f patotski_zaqlick-app

# Check stack and service status
docker stack ls
docker service ls
docker service ps patotski_zaqlick-app

# Clean up unused images
docker image prune -f
```

## Deployment Flow

1. Source repo triggers `repository_dispatch` event with `version_var` and `image_tag`
2. GitHub Actions updates `.env` with new version
3. Config files generated from `**/application.yaml.template` using `envsubst`
4. Files SCP'd to EC2, then SSH to pull images and restart services

## Key Files

- `.env` - Image versions (auto-updated by CI/CD)
- `docker-compose.yml` - Service definitions with Traefik labels
- `.github/workflows/update-and-deploy.yml` - Deployment workflow
- `zaqlick-app/application.yaml.template` - App config template (uses GitHub secrets)
- `social-data-app/application.yaml.template` - Parser config template

## Version Variables

| Variable | Service |
|----------|---------|
| `ZAQLICK_APP_VERSION` | zaqlick-app |
| `ZAQLICK_SOCIAL_DATA_PARSER_VERSION` | zaqlick-social-data-parser |
| `TRAEFIK_VERSION` | traefik |

## Triggering Deployments

From other repos, use `repository_dispatch`:
```yaml
- uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.DEPLOY_TOKEN }}
    repository: patotski/docker-compose
    event-type: update-image
    client-payload: '{"version_var": "ZAQLICK_APP_VERSION", "image_tag": "v1.0.0"}'
```

## Network

All services on `traefik_web` overlay network (Docker Swarm). Traefik routes by hostname using labels under `deploy.labels`.

## Healthchecks

The Spring Boot app images (`zaqlick-app`, `zaqlick-social-data-parser`) are built with Paketo buildpacks using a distroless/tiny base image — **no `/bin/sh`**. This means `CMD-SHELL` healthchecks will always fail with `exec: "/bin/sh": no such file or directory`.

Healthchecks are intentionally omitted from these services. If healthcheck support is needed in the future, add `curl` or a dedicated health binary to the app's Dockerfile.

## Rolling Update Order

`zaqlick-app` uses `update_config.order: start-first` (and `rollback_config.order: start-first`) for true zero-downtime rolling updates.

**Why this works:** With 3 nodes, 2 replicas, and `max_replicas_per_node: 1`, there is always a free node available to start the new container before stopping the old one.

**If a node is removed** and you're back to 2 nodes, `start-first` will cause a scheduling deadlock — switch to `stop-first` until a 3rd node is available again.

## Swarm Nodes

| Host alias     | Role    |
|----------------|---------|
| `zaqlick-app`  | Manager |
| `zaqlick-app-2`| Worker  |
| `zaqlick-app-3`| Worker  |

The workflow provisions all 3 nodes via `EC2_HOST`, `EC2_HOST_2`, `EC2_HOST_3` GitHub Actions variables.
