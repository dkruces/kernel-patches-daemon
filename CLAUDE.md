# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Kernel Patches Daemon (KPD) is a Python service that bridges Patchwork (Linux kernel patch management) with GitHub repositories for automated CI testing. It watches Patchwork for new patch series and syncs them as pull requests in a GitHub repository, then reports CI results back to Patchwork and optionally emails patch authors.

Originally developed at Meta for BPF subsystem testing in the Linux Kernel.

## Development Commands

### Environment Setup
```bash
# Install poetry if not already installed
pip install --user poetry

# Setup virtualenv and install dependencies
python -m venv .venv
poetry install
```

### Testing
```bash
# Run all tests
poetry run python -m unittest

# Run a specific test file
poetry run python -m unittest tests.test_branch_worker

# Run a specific test case
poetry run python -m unittest tests.test_branch_worker.TestBranchWorker.test_slugify_context
```

### Code Formatting and Linting
```bash
# Format code with black (required before committing)
poetry run black .

# Run flake8 linting
poetry run flake8
```

### Running the Daemon
```bash
# Start the daemon (default action)
poetry run python -m kernel_patches_daemon --config <config_path> --label-colors configs/labels.json

# Purge all PRs and branches (destructive)
poetry run python -m kernel_patches_daemon --config <config_path> --action purge

# Enable OpenTelemetry console metrics (JSON output to stdout)
# Note: Console metrics are disabled by default to reduce log clutter
poetry run python -m kernel_patches_daemon --config <config_path> --label-colors configs/labels.json --enable-console-metrics

# Use external script for custom metrics processing
# The --metric-logger expects an executable script that receives JSON metrics via stdin
poetry run python -m kernel_patches_daemon --config <config_path> --label-colors configs/labels.json --metric-logger /path/to/metrics_processor.sh
```

### Docker
```bash
# Build image
docker-compose build

# Start service
docker-compose up

# Use pre-built image
docker pull ghcr.io/kernel-patches/kernel-patches-daemon:latest
```

## Architecture

### Core Components

**KernelPatchesDaemon** (`daemon.py`)
- Entry point that runs KernelPatchesWorker in a loop with configurable delay (default 120s)
- Handles graceful shutdown on SIGTERM/SIGINT
- Implements crash recovery with configurable retry limits

**GithubSync** (`github_sync.py`)
- Main orchestrator for the sync cycle
- Manages multiple BranchWorker instances (one per configured branch)
- Fetches relevant subjects from Patchwork
- Maps patch tags to target branches
- Coordinates PR creation/updates across workers
- Closes PRs when series switch target branches

**BranchWorker** (`branch_worker.py`)
- Manages a single GitHub repository branch
- Clones/fetches upstream and CI repos
- Applies patch series using `git am --3way`
- Creates/updates/closes pull requests
- Syncs CI workflow results to Patchwork as checks
- Sends email notifications to patch authors
- Expires stale branches and user-created PRs
- Handles merge conflicts by creating placeholder PRs with merge-conflict label

**Patchwork** (`patchwork.py`)
- Client for Patchwork REST API with retry logic
- Fetches relevant patch series based on search patterns and lookback window
- Tracks series state (new, under-review, accepted, superseded, etc.)
- Creates checks on Patchwork for CI results
- Implements caching with TTL

**Config** (`config.py`)
- Configuration version 3 (only supported version)
- Supports both GitHub OAuth tokens and GitHub App authentication
- tag_to_branch_mapping: maps Patchwork tags to target branch order (tries branches in order until apply succeeds)
- Optional mirror support for faster git clones
- Email configuration for notifications

### Key Workflows

**Sync Cycle** (sync_patches in github_sync.py):
1. Refresh all branch workers: fetch repos, get current PRs, sync upstream to target branch
2. Fetch relevant subjects from Patchwork
3. For each subject:
   - Get latest series and its tags
   - Map tags to target branches
   - Try applying to branches in order until success
   - Create/update PR for successful branch
   - Sync CI checks back to Patchwork
   - Close PRs on other branches for same series
4. Update old subjects that weren't in fresh fetch
5. Expire stale branches and user PRs

**PR Branch Naming**: `series/<series_id>=><target_branch>`
- Example: `series/123456=>bpf` for series 123456 targeting bpf branch
- Parsed by parse_pr_ref() in branch_worker.py

**CI Status Reporting**:
- Fetches GitHub Actions workflow runs for PR head SHA
- Submits individual job results as separate Patchwork checks
- Uses slugified context names (dots replaced with underscores)
- Adds version-specific labels to PRs: V1-ci-pass, V2-ci-fail, etc.
- Sends email on first status label for a patch version

**Mirror Support** (optional):
- Speeds up git clones by using `--reference-if-able` to local mirror
- Falls back to mirror_fallback_repo if primary mirror doesn't exist
- Configured per-branch with mirror_dir and mirror_fallback_repo

### Important Patterns

**Subject vs Series**: A Subject is the normalized patch subject line. Multiple Series (versions) can share the same Subject. KPD creates one PR per Subject, updating it as new Series versions arrive.

**Branch Selection**: When multiple target branches are configured for a tag, KPD tries applying patches in order. If an existing PR without merge conflicts exists on one branch, that branch is "sticky" and retried first.

**Rate Limiting**: BranchWorker checks GitHub rate limit before sync. If remaining tokens < 1000, can_do_sync() returns False to skip that branch.

**Email Notifications**: Only sent when CI status label changes (first time or pass<->fail). Uses curl for SMTP to support HTTP proxies. Submitter allowlist controls rollout.

## Configuration Notes

- Config files in `configs/` directory (see kpd.json example)
- GitHub App auth is preferred over personal tokens
- App needs: Contents (write), Pull Requests (write), Workflows (read/write)
- search_patterns in Patchwork config filters which patches to track
- lookback controls how far back to search (in days)

## Code Style

- Black formatter is enforced (must run before committing)
- Python 3.10+ required
- Type hints present but marked `# pyre-unsafe` in most files
- OpenTelemetry metrics throughout for observability
  - Console metrics disabled by default (use `--enable-console-metrics` to enable)
  - Metrics exported via `--metric-logger` to executable scripts for custom processing
  - Metrics include: sync timing, git operations, PR lifecycle, API requests, errors

## Contributing

- All commits require Signed-off-by (Developer Certificate of Origin)
- Fork from main branch
- Add tests for new functionality
- Ensure tests pass and code is formatted with black
