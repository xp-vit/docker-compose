# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Docker Compose deployment orchestration for multiple microservices on EC2. Uses Traefik as reverse proxy with automatic SSL, and GitHub Actions for CI/CD.

## Services

- **traefik** - Reverse proxy with Let's Encrypt SSL (ports 80, 443)
- **zaqlick-app** - Main application at `app2.zaqlick.com` (Spring Boot)
- **zaqlick-social-data-parser** - Data parser at `data.patotski.com` (Spring Boot)

## Common Commands

```bash
# Start all services
docker compose up -d

# Restart a specific service
docker compose up -d --no-deps --force-recreate zaqlick-app

# Pull latest images and redeploy
docker compose pull && docker compose up -d

# View logs
docker compose logs -f zaqlick-app

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

All services on `traefik_web` bridge network. Traefik routes by hostname using Docker labels.
