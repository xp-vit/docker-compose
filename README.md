# docker-compose

Deployment configuration repository for multiple services.

## How It Works

Image versions are stored in the `.env` file and referenced in `docker-compose.yml` using variable interpolation:

```yaml
image: xpvit/zaqlick-social-data-parser:${ZAQLICK_SOCIAL_DATA_PARSER_VERSION:-latest}
```

## Triggering Deployment from Other Repositories

When a build finishes in another repository, you can trigger this deployment workflow using GitHub's `repository_dispatch` event.

### Step 1: Create a Personal Access Token (PAT)

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate a new token with `repo` scope
3. Save this token as a secret in your source repository (e.g., `DEPLOY_TOKEN`)

### Step 2: Add Trigger Step to Source Repository Workflow

Add this step at the end of your build workflow in the source repository:

```yaml
- name: Trigger deployment
  uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.DEPLOY_TOKEN }}
    repository: patotski/docker-compose
    event-type: update-image
    client-payload: |
      {
        "version_var": "ZAQLICK_SOCIAL_DATA_PARSER_VERSION",
        "image_tag": "v1.2.3"
      }
```

### Alternative: Using curl

You can also trigger the workflow using curl:

```bash
curl -X POST \
  -H "Accept: application/vnd.github+json" \
  -H "Authorization: Bearer YOUR_GITHUB_TOKEN" \
  https://api.github.com/repos/patotski/docker-compose/dispatches \
  -d '{"event_type":"update-image","client_payload":{"version_var":"ZAQLICK_SOCIAL_DATA_PARSER_VERSION","image_tag":"v1.2.3"}}'
```

### Payload Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `version_var` | Environment variable name in `.env` | `ZAQLICK_SOCIAL_DATA_PARSER_VERSION` |
| `image_tag` | New image tag/version | `v1.2.3`, `latest`, or `${{ github.sha }}` |

### Available Version Variables

| Variable | Service | Image |
|----------|---------|-------|
| `ZAQLICK_SOCIAL_DATA_PARSER_VERSION` | zaqlick-social-data-parser | xpvit/zaqlick-social-data-parser |
| `ZAQLICK_APP_VERSION` | zaqlick-app | api.repoflow.io/xpvitblr-504/zaqlick-app/zaqlick-app |
| `TRAEFIK_VERSION` | traefik | traefik |

## Manual Trigger

You can also manually trigger the workflow from the GitHub Actions tab using the "Run workflow" button with the required inputs.

## Services

- **traefik**: Reverse proxy and load balancer with automatic SSL
- **zaqlick-social-data-parser**: Social data parsing service
- **zaqlick-app**: Main application
