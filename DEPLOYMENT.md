# Deployment Guide

This repository is configured to deploy the Case Support WordPress plugin to the UAT environment via GitHub Actions.

## Workflow
- **Workflow file:** `.github/workflows/deploy.yml`
- **Triggers:**
  - Pushes to the `main` or `work` branches.
  - Manual runs via **Run workflow** (workflow_dispatch).
- **Environment:** `UAT` (configured in GitHub with required secrets).

## Required secrets (UAT environment)
- `SSH_PRIVATE_KEY`: Private key with SSH access to the UAT host.
- `REMOTE_HOST`: Hostname or IP for the UAT server.
- `REMOTE_USER`: SSH username with write access to the plugin directory.

## Deployment details
- Source directory synced: repository root (configured via `SOURCE_DIR=.`). Update `SOURCE_DIR` if the plugin lives in a subdirectory.
- Remote path: `/home/uathub/htdocs/uathub.udwresourcecenter.org/wp-content/plugins/case-support/`
- Files are transferred with `rsync` using `--delete` to keep the remote directory in sync.
- The workflow excludes `.git/` and `.github/` from deployment by default.

## Notes
- The workflow ensures the remote path exists before syncing.
- Host keys are added automatically via `ssh-keyscan` for the provided host.
- Update the workflow `paths` filter if additional deployment-relevant files are added outside `case-support/`.
