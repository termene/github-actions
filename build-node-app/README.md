# Build Node.js App - Composite Action

Reusable GitHub Action for testing, linting, building and packaging Node.js applications.

## Features

- ✅ Node.js setup with version management
- ✅ NPM cache for faster builds
- ✅ Dependency installation
- ✅ Configurable linting
- ✅ Configurable testing
- ✅ Build execution
- ✅ Artifact packaging and upload
- ✅ Customizable file inclusion

## Usage

### Basic Usage

```yaml
- name: Checkout
  uses: actions/checkout@v4

- name: Build and test
  uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
```

### With Custom Configuration

```yaml
- name: Build and test
  uses: your-org/github-actions/build-node-app@v1
  with:
    node_version: '20.10.0'
    app_name: my-app
    commit_sha: ${{ github.sha }}
    run_tests: 'true'
    run_lint: 'true'
    test_command: 'npm run test:ci'
    lint_command: 'npm run lint:strict'
    artifact_files: 'dist/ package.json .env.example'
    artifact_retention_days: '7'
```

### Skip Tests or Linting

```yaml
- name: Build only (no tests/lint)
  uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
    run_tests: 'false'
    run_lint: 'false'
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `node_version` | Node.js version to use | No | `22.21.0` |
| `app_name` | Application name for artifact | Yes | - |
| `commit_sha` | Commit SHA for artifact naming | Yes | - |
| `run_tests` | Whether to run tests | No | `true` |
| `run_lint` | Whether to run linting | No | `true` |
| `test_command` | Test command to run | No | `npm run test:cov --if-present` |
| `lint_command` | Lint command to run | No | `npm run lint --if-present` |
| `build_command` | Build command to run | No | `npm run build` |
| `artifact_files` | Files to include (space-separated) | No | `dist/ package.json package-lock.json ecosystem.config.js` |
| `artifact_retention_days` | Days to retain artifact | No | `30` |

## Outputs

| Output | Description |
|--------|-------------|
| `artifact_name` | Name of the uploaded artifact |
| `artifact_path` | Path to the artifact file |

## Examples

### Full CI Pipeline

```yaml
name: CI

on: [push, pull_request]

env:
  APP_NAME: my-app

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build and test
        id: build
        uses: your-org/github-actions/build-node-app@v1
        with:
          app_name: ${{ env.APP_NAME }}
          commit_sha: ${{ github.sha }}
      
      - name: Show artifact info
        run: |
          echo "Artifact name: ${{ steps.build.outputs.artifact_name }}"
          echo "Artifact path: ${{ steps.build.outputs.artifact_path }}"
```

### Different Node Versions

```yaml
strategy:
  matrix:
    node-version: [18.x, 20.x, 22.x]

steps:
  - uses: actions/checkout@v4
  
  - uses: your-org/github-actions/build-node-app@v1
    with:
      node_version: ${{ matrix.node-version }}
      app_name: my-app
      commit_sha: ${{ github.sha }}
```

### Custom Artifact Contents

```yaml
- uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
    artifact_files: |
      dist/
      package.json
      .env.example
      config/
      scripts/
    artifact_retention_days: '30'
```

### Production Build (no tests)

```yaml
- uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
    run_tests: 'false'
    run_lint: 'true'
    build_command: 'npm run build:prod'
```

## Requirements

### In your repository:
- `package.json` with appropriate scripts
- `package-lock.json` for dependency locking
- TypeScript config (if using TypeScript)

### Expected `package.json` scripts:
```json
{
  "scripts": {
    "build": "tsc",
    "lint": "eslint .",
    "test": "jest",
    "test:cov": "jest --coverage"
  }
}
```

## Artifact Structure

The action creates a `.tar.gz` artifact containing the specified files:

```
my-app-abc123def.tar.gz
├── dist/
│   ├── index.js
│   └── ...
├── package.json
├── package-lock.json
└── ecosystem.config.js
```

## Use Cases

### 1. Development Workflow
```yaml
- uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
    run_tests: 'true'
    run_lint: 'true'
```

### 2. Production Build
```yaml
- uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
    run_tests: 'false'  # Tests run in separate job
    build_command: 'npm run build:prod'
    artifact_retention_days: '90'
```

### 3. Quick Build (CI/CD speed optimization)
```yaml
- uses: your-org/github-actions/build-node-app@v1
  with:
    app_name: my-app
    commit_sha: ${{ github.sha }}
    run_tests: 'false'
    run_lint: 'false'
    artifact_retention_days: '1'
```

## Integration with Deployment

Use outputs to pass artifact info to deployment jobs:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.build.outputs.artifact_name }}
    steps:
      - uses: actions/checkout@v4
      - id: build
        uses: your-org/github-actions/build-node-app@v1
        with:
          app_name: my-app
          commit_sha: ${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: ${{ needs.build.outputs.artifact-name }}
      # ... deployment steps
```

## Notes

- The action uses `npm ci` for reproducible builds
- Cache is automatically enabled for faster subsequent builds
- `--if-present` flags allow missing scripts to not fail the build
- All commands run with `bash` shell

