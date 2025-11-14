# Deploy Node.js App - Composite Action

Reusable GitHub Action for deploying Node.js applications via SSH with artifact extraction and PM2 management.

## Features

- ✅ SSH deployment with key authentication
- ✅ Git checkout (commits or tags)
- ✅ Artifact extraction with `.env` preservation
- ✅ Node.js version management via `n`
- ✅ Automatic dependency installation
- ✅ PM2 process management (restart/reload/skip)
- ✅ Automatic cleanup

## Usage

### Basic Deployment (DEV - no PM2 restart)

```yaml
- name: Deploy to DEV
  uses: your-org/github-actions/deploy-node-app@v1
  with:
    ssh_host: dev.example.com
    ssh_username: github
    ssh_key: ${{ secrets.SSH_KEY }}
    app_name: my-app
    artifact_path: /tmp/my-app-${{ github.sha }}.tar.gz
    commit_sha: ${{ github.sha }}
    pm2_action: skip
```

### Production Deployment (with PM2 restart)

```yaml
- name: Deploy to PROD
  uses: your-org/github-actions/deploy-node-app@v1
  with:
    ssh_host: prod.example.com
    ssh_username: github
    ssh_key: ${{ secrets.SSH_KEY }}
    app_name: my-app
    artifact_path: /tmp/my-app-${{ github.sha }}.tar.gz
    commit_sha: ${{ github.ref_name }}
    use_tags: 'true'
    pm2_action: restart
    pm2_app_name: my-app
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `ssh_host` | SSH host to deploy to | Yes | - |
| `ssh_username` | SSH username | No | `github` |
| `ssh_key` | SSH private key | Yes | - |
| `app_name` | Application name | Yes | - |
| `deploy_path` | Base deployment path | No | `/home/github` |
| `artifact_path` | Path to artifact on server | Yes | - |
| `commit_sha` | Commit SHA or tag name | Yes | - |
| `use_tags` | Fetch tags instead of commits | No | `false` |
| `node_version` | Node.js version (via `n`) | No | `22.21.0` |
| `pm2_action` | PM2 action: `restart`, `reload`, `skip` | No | `skip` |
| `pm2_app_name` | PM2 application name | No | - |

## Requirements

### On the deployment server:
- Git repository initialized in deploy path
- `n` (Node version manager) installed
- `pm2` installed globally (if using PM2)
- SSH key authorized for the user

### In your workflow:
- Artifact already uploaded to server (via SCP or similar)
- Repository checked out if using repository action path

## Examples

### Deploy to multiple environments

```yaml
jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: your-org/github-actions/deploy-node-app@v1
        with:
          ssh_host: dev.example.com
          ssh_key: ${{ secrets.DEV_SSH_KEY }}
          app_name: my-app
          artifact_path: /tmp/my-app.tar.gz
          commit_sha: ${{ github.sha }}
          pm2_action: skip

  deploy-prod:
    runs-on: ubuntu-latest
    needs: deploy-dev
    steps:
      - uses: actions/checkout@v4
      - uses: your-org/github-actions/deploy-node-app@v1
        with:
          ssh_host: prod.example.com
          ssh_key: ${{ secrets.PROD_SSH_KEY }}
          app_name: my-app
          artifact_path: /tmp/my-app.tar.gz
          commit_sha: ${{ github.ref_name }}
          use_tags: 'true'
          pm2_action: restart
          pm2_app_name: my-app
```

### Use from another repository

To use this action from another repository:
```yaml
- uses: your-org/github-actions/deploy-node-app@v1
  with:
    # ... inputs
```

## PM2 Actions

| Action | Description | Use Case |
|--------|-------------|----------|
| `skip` | Don't touch PM2 | DEV environments, manual control |
| `restart` | Hard restart (stop + start) | PROD deployments, config changes |
| `reload` | Zero-downtime reload | PROD updates, no downtime needed |

## Notes

- The action preserves `.env` files and other non-archived files
- Git reset is always hard, be careful with uncommitted changes
- Artifact cleanup happens automatically
- Node version must be installed via `n` beforehand

