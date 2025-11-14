# Reusable GitHub Actions

This directory contains reusable composite actions for CI/CD workflows.

## Available Actions

### 1. `build-node-app`
Build, test, lint and package Node.js applications.

**Use case:** CI pipelines, artifact creation

```yaml
- uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
```

[📖 Full Documentation](./build-node-app/README.md)

---

### 2. `setup-ssh`
Setup SSH keys and manage known_hosts.

**Use case:** Prepare SSH for deployments, SCP transfers

```yaml
- uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.SSH_KEY }}
    known_hosts: dev.example.com,prod.example.com
```

[📖 Full Documentation](./setup-ssh/README.md)

---

### 3. `deploy-node-app`
Deploy Node.js applications via SSH with PM2 management.

**Use case:** Deployment to DEV/PROD servers

```yaml
- uses: your-org/github-actions/deploy-node-app@v1
  with:
    ssh_host: example.com
    ssh_key: ${{ secrets.SSH_KEY }}
    app_name: my-app
    artifact_path: /tmp/my-app.tar.gz
    commit_sha: ${{ github.sha }}
    pm2_action: restart
```

[📖 Full Documentation](./deploy-node-app/README.md)

---

## Complete CI/CD Example

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: ["**"]

env:
  APP_NAME: my-app

jobs:
  # Step 1: Build and test
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and test
        uses: termene/github-actions/build-node-app@v1
        with:
          app_name: ${{ env.APP_NAME }}
          commit_sha: ${{ github.sha }}

  # Step 2: Deploy to DEV
  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ env.APP_NAME }}-${{ github.sha }}
      
      - name: Setup SSH
        uses: termene/github-actions/setup-ssh@v1
        with:
          ssh_key: ${{ secrets.SSH_KEY }}
          known_hosts: dev.server
      
      - name: Copy artifact
        run: |
          scp -i ~/.ssh/deploy_key ${{ env.APP_NAME }}-*.tar.gz user@dev.server:/tmp/
      
      - name: Deploy
        uses: termene/github-actions/deploy-node-app@v1
        with:
          ssh_host: dev.server
          ssh_key: ${{ secrets.SSH_KEY }}
          app_name: ${{ env.APP_NAME }}
          artifact_path: /tmp/${{ env.APP_NAME }}-${{ github.sha }}.tar.gz
          commit_sha: ${{ github.sha }}
          pm2_action: skip

  # Step 3: Deploy to PROD (only on tags)
  deploy-prod:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Download artifact
        uses: actions/download-artifact@v4
        with:
          name: ${{ env.APP_NAME }}-${{ github.sha }}
      
      - name: Deploy
        uses: termene/github-actions/deploy-node-app@v1
        with:
          ssh_host: prod.server
          ssh_key: ${{ secrets.PROD_SSH_KEY }}
          app_name: ${{ env.APP_NAME }}
          artifact_path: /tmp/${{ env.APP_NAME }}-${{ github.sha }}.tar.gz
          commit_sha: ${{ github.ref_name }}
          use_tags: 'true'
          pm2_action: restart
          pm2_app_name: my-app
```

## Reusing in Other Repositories

```yaml
- uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
```

### Option: Git Submodule in your TypeScript projects
```bash
# In each repo
git submodule add https://github.com/your-org/github-actions .github/actions
```

## Benefits

✅ **DRY** - Don't repeat deployment logic  
✅ **Consistency** - Same CI/CD across all projects  
✅ **Maintainability** - Fix bugs once, benefit everywhere  
✅ **Flexibility** - Configurable for different use cases  
✅ **Documentation** - Each action has detailed README  

## Action Composition

These actions are designed to work together:

```
build-node-app
    ↓ (produces artifact)
download-artifact
    ↓ (get artifact)
deploy-node-app
    ↓ (deploys artifact)
PM2 running app
```

## Best Practices

1. **Pin versions** when using from external repos:
   ```yaml
   uses: your-org/github-actions/build-node-app@v1.2.3
   ```

2. **Use outputs** to pass data between actions:
   ```yaml
   - id: build
     uses: your-org/github-actions/build-node-app@v1.2.3
   - run: echo ${{ steps.build.outputs.artifact_name }}
   ```

3. **Keep secrets secure** - Never hardcode, always use `${{ secrets.NAME }}`

4. **Test locally** using [act](https://github.com/nektos/act):
   ```bash
   act -j build
   ```

## Contributing

When adding new actions:
1. Create a new action directory
2. Add `action.yml` with clear inputs/outputs
3. Add comprehensive `README.md`
4. Update this main README with the new action
5. Test in a real workflow

## Troubleshooting

### Action not found
- Ensure you've checked out the repo: `uses: actions/checkout@v4`
- Verify the path: `termene/github-actions/[action-name]`

### SSH connection failed
- Check SSH key is valid
- Verify host is in known_hosts
- Ensure server has git repository initialized

### PM2 not found
- Install PM2 globally on target server: `npm install -g pm2`
- Verify PM2 is in PATH

### Artifact download failed
- Check artifact was uploaded in previous job
- Verify artifact name matches exactly
- Ensure artifact hasn't expired

