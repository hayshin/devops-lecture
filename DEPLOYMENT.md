# Deployment Notes

This repository builds a static Vite site into an nginx container image and publishes it to GitHub Container Registry.

## Image Tags

The CI/CD workflow publishes images to:

```text
ghcr.io/<owner>/<repo>
```

Pushes to `main` publish a commit SHA tag and a branch tag. Prefer deploying immutable SHA tags rather than branch tags.

## Rollback

Redeploy the last known-good image SHA in your hosting platform, for example:

```bash
docker pull ghcr.io/<owner>/<repo>:<previous-good-sha>
docker run --rm -p 8080:8080 ghcr.io/<owner>/<repo>:<previous-good-sha>
curl -fsS http://127.0.0.1:8080/healthz
```

For Kubernetes-based deployments, update the image back to the previous-good SHA and verify rollout health:

```bash
kubectl set image deployment/devops-lecture site=ghcr.io/<owner>/<repo>:<previous-good-sha>
kubectl rollout status deployment/devops-lecture
```

## Docker Compose Production

Use `docker-compose.prod.yml` on a host where ports `80` and `443` are available. Configure the domain and image tag in an env file:

```bash
cp .env.production.example .env.production
docker compose --env-file .env.production -f docker-compose.prod.yml pull
docker compose --env-file .env.production -f docker-compose.prod.yml up -d
curl -fsS https://example.com/healthz
```

`https-portal` terminates TLS and proxies to the internal `nginx` app service on port `8080`.

## GitHub Actions Server Deploy

Pushes to `main` or `master` publish the image to GHCR, then deploy the same immutable commit SHA tag on the server with Docker Compose.

Configure these repository secrets:

```text
DEPLOY_HOST=your.server.example.com
DEPLOY_USER=deploy
DEPLOY_SSH_KEY=<private ssh key for DEPLOY_USER>
DEPLOY_DOMAINS=example.com -> http://nginx:8080
```

Optional secrets:

```text
DEPLOY_PORT=22
DEPLOY_DIR=/opt/devops-lecture
HTTPS_PORTAL_STAGE=production
CLIENT_MAX_BODY_SIZE=10M
DEPLOY_GHCR_USER=<github username or bot>
DEPLOY_GHCR_TOKEN=<classic PAT or fine-grained token with package read access>
```

The server needs Docker Compose v2 installed and `DEPLOY_USER` must be able to run Docker commands.
