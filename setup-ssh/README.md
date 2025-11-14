# Setup SSH - Composite Action

Reusable GitHub Action for setting up SSH keys and managing known_hosts.

## Features

- ✅ Creates SSH directory with proper permissions
- ✅ Sets up SSH private key
- ✅ Adds multiple hosts to known_hosts automatically
- ✅ Idempotent - safe to run multiple times
- ✅ Supports comma-separated host lists
- ✅ Uses ssh-keyscan for secure host key verification

## Usage

### Basic Usage

```yaml
- name: Setup SSH
  uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.SSH_KEY }}
    known_hosts: example.com
```

### Multiple Hosts

```yaml
- name: Setup SSH
  uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.SSH_KEY }}
    known_hosts: dev.example.com,prod.example.com,staging.example.com
```

### Custom SSH Key Path

```yaml
- name: Setup SSH
  uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.SSH_KEY }}
    known_hosts: example.com
    ssh_key_path: ~/.ssh/custom_key
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `ssh_key` | SSH private key content | Yes | - |
| `ssh_key_path` | Path to store SSH key | No | `~/.ssh/deploy_key` |
| `known_hosts` | Comma-separated list of hosts | Yes | - |
| `ssh_dir` | SSH directory path | No | `~/.ssh` |

## Examples

### Deploy Workflow

```yaml
name: Deploy

on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup SSH
        uses: your-org/github-actions/setup-ssh@v1
        with:
          ssh_key: ${{ secrets.DEPLOY_KEY }}
          known_hosts: deploy.example.com
      
      - name: Deploy via SCP
        run: |
          scp -i ~/.ssh/deploy_key artifact.tar.gz user@deploy.example.com:/tmp/
      
      - name: Execute deployment
        run: |
          ssh -i ~/.ssh/deploy_key user@deploy.example.com 'bash deploy.sh'
```

### Multi-Environment Setup

```yaml
- name: Setup SSH for all environments
  uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.SSH_KEY }}
    known_hosts: |
      dev.example.com,
      staging.example.com,
      prod.example.com,
      backup.example.com

- name: Deploy to dev
  run: scp -i ~/.ssh/deploy_key app.tar.gz user@dev.example.com:/app/

- name: Deploy to prod
  run: scp -i ~/.ssh/deploy_key app.tar.gz user@prod.example.com:/app/
```

### Custom Key for Different Servers

```yaml
- name: Setup SSH for dev
  uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.DEV_SSH_KEY }}
    known_hosts: dev.example.com
    ssh_key_path: ~/.ssh/dev_key

- name: Setup SSH for prod
  uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.PROD_SSH_KEY }}
    known_hosts: prod.example.com
    ssh_key_path: ~/.ssh/prod_key

- name: Deploy to dev
  run: scp -i ~/.ssh/dev_key app.tar.gz user@dev.example.com:/app/

- name: Deploy to prod
  run: scp -i ~/.ssh/prod_key app.tar.gz user@prod.example.com:/app/
```

### GitHub Actions Self-Hosted Runner

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, linux]
    steps:
      - name: Setup SSH
        uses: your-org/github-actions/setup-ssh@v1
        with:
          ssh_key: ${{ secrets.SSH_KEY }}
          known_hosts: internal.server.local
      
      - name: Internal deployment
        run: |
          ssh -i ~/.ssh/deploy_key internal@internal.server.local 'cd /app && git pull'
```

## How It Works

1. **Creates SSH directory** (`~/.ssh`) with proper permissions (700)
2. **Writes SSH key** to specified path with secure permissions (600)
3. **Adds hosts to known_hosts** using `ssh-keyscan -H` for hashed entries
4. **Checks existing entries** to avoid duplicates (idempotent)

## Security Notes

### SSH Key Storage
- Always use GitHub Secrets for SSH keys
- Never commit SSH keys to repository
- Use different keys for different environments

### Known Hosts
- Uses `ssh-keyscan -H` for hashed host keys
- Prevents MITM attacks by verifying host fingerprints
- Safe to run multiple times (checks for existing entries)

### Permissions
- SSH directory: `700` (owner only)
- SSH key file: `600` (owner read/write only)
- Known hosts file: `644` (world-readable, standard practice)

## Troubleshooting

### Permission denied (publickey)
```
Permission denied (publickey).
```
**Solution:** Verify the SSH key in secrets matches the authorized_keys on the server

### Host key verification failed
```
Host key verification failed.
```
**Solution:** Ensure the host is in the `known_hosts` input parameter

### Key already exists message
```
ℹ️  SSH key already exists at ~/.ssh/deploy_key
```
**Note:** This is informational - the action is idempotent and won't overwrite existing keys

### Connection timeout
```
ssh: connect to host example.com port 22: Connection timed out
```
**Solution:** Check network connectivity and firewall rules

## Integration Examples

### With SCP Transfer

```yaml
- uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.SSH_KEY }}
    known_hosts: server.com

- run: scp -i ~/.ssh/deploy_key file.tar.gz user@server.com:/tmp/
```

### With SSH Commands

```yaml
- uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.SSH_KEY }}
    known_hosts: server.com

- run: |
    ssh -i ~/.ssh/deploy_key user@server.com << 'EOF'
      cd /app
      tar -xzf /tmp/file.tar.gz
      pm2 restart app
    EOF
```

### With appleboy/ssh-action

```yaml
- uses: your-org/github-actions/setup-ssh@v1
  with:
    ssh_key: ${{ secrets.SSH_KEY }}
    known_hosts: server.com

- uses: appleboy/ssh-action@v1
  with:
    host: server.com
    username: user
    key: ${{ secrets.SSH_KEY }}
    script: |
      echo "Deployment started"
      ./deploy.sh
```

## Best Practices

1. **Use descriptive secret names**: `PROD_SSH_KEY`, `DEV_SSH_KEY`
2. **Separate keys per environment**: Don't reuse the same key
3. **Rotate keys regularly**: Update secrets periodically
4. **Limit key permissions**: Use read-only keys where possible
5. **Audit key usage**: Review GitHub Actions logs regularly

## Output Files

After running this action:
- `~/.ssh/` - SSH directory (or custom path)
- `~/.ssh/deploy_key` - SSH private key (or custom path)
- `~/.ssh/known_hosts` - Known hosts file with added hosts

## Notes

- Action is **idempotent** - safe to run multiple times
- Existing SSH keys are preserved (won't overwrite)
- Hosts already in known_hosts are skipped
- Works on Linux, macOS, and Windows (with bash)

