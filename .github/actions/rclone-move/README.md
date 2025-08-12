# rclone-move (GitHub Action)

Move files or directories using rclone. Supports optional rclone configuration, include/exclude filtering via a generated filter file, dry-run mode, and cleanup of empty source directories.

## Overview

This composite action wraps `rclone move` with sensible defaults and a simple interface for CI workflows on Ubuntu or self-hosted Linux runners.

- Installs `rclone` via `apt` if missing
- Accepts inline rclone config (pass secrets) or uses an existing config on the runner
- Applies include/exclude patterns using a temporary `--filter-from` file for predictable precedence
- Supports parallel transfers, retries, size-only comparison, dry-run, and deletion of empty source dirs

## Requirements

- GitHub-hosted Ubuntu runner or self-hosted Linux runner with `apt` available. The action refreshes package indexes and installs rclone via `apt` if missing.
- Network access to your destination remote (e.g. cloud storage) and any required credentials.
- Optional: an rclone config provided via `${{ secrets.RCLONE_CONFIG }}` (full file contents), or a pre-existing config at `$HOME/.config/rclone/rclone.conf`.

## Inputs

| Name | Required | Default | Description |
|------|----------|---------|-------------|
| `source_path` | yes | — | Absolute path to the source file/dir on the runner. |
| `dest_path` | yes | — | rclone destination, e.g. `remote:bucket/prefix` or an absolute local path. |
| `transfers` | no | `"6"` | Number of parallel transfers (`--transfers`). Positive integer. |
| `retries` | no | `"8"` | Max retries for failed transfers (`--retries`). Non-negative integer. |
| `size_only` | no | `"true"` | Use `--size-only` to decide if a file needs transferring. |
| `dry_run` | no | `"false"` | If `true`, add `--dry-run` (no data moved). |
| `delete_empty_src_dirs` | no | `"true"` | Add `--delete-empty-src-dirs` after moving. |
| `include_pattern` | no | `""` | Comma-separated include patterns (see Filter handling). |
| `exclude_pattern` | no | `""` | Comma-separated exclude patterns (see Filter handling). |
| `rclone_config` | no | `""` | Full contents of an rclone config file. Written to `$HOME/.config/rclone/rclone.conf` if provided. |

## Filter handling (include/exclude)

The action converts `include_pattern` and `exclude_pattern` into a temporary filter file and passes it with `--filter-from` for predictable rule order.

- Patterns come from comma-separated lists. Whitespace around entries is trimmed.
- When includes are present:
  - The filter file starts with `+ /**/` to allow directory traversal.
  - Each include adds a `+ <pattern>` line.
  - Each exclude adds a `- <pattern>` line.
  - Finally, `- *` excludes everything else (allow-list behavior).
- When only excludes are present, only `- <pattern>` lines are written (block-list behavior).
- Patterns use rclone’s filtering syntax. Quote globs in YAML (e.g. `'*/*.txt'`) to avoid YAML parsing surprises.

Notes:
- Commas separate patterns; literal commas in a pattern are not supported.
- Patterns are matched relative to `source_path`. Consult rclone docs for advanced rules if needed.

## Examples

Basic move to an S3-compatible remote using a secret rclone config:

```yaml
name: Move Artifacts
on:
  workflow_dispatch:

jobs:
  move:
    runs-on: [self-hosted, Linux, X64]
    steps:
      - uses: actions/checkout@v4
      - name: Move artifacts to object storage
        uses: EODC-GmbH/github-actions/.github/actions/rclone-move@main
        with:
          source_path: ${{ github.workspace }}/dist
          dest_path: s3:my-bucket/releases/${{ github.sha }}
          include_pattern: '*.whl,*.tar.gz'
          exclude_pattern: '*.log,**/tmp/**'
          transfers: '8'
          retries: '10'
          rclone_config: ${{ secrets.RCLONE_CONFIG }}

```

Dry run for testing filter rules:

```yaml
name: Rclone Dry Run
on:
  workflow_dispatch:

jobs:
  dryrun:
    runs-on: [self-hosted, Linux, X64]
    steps:
      - uses: actions/checkout@v4
      - name: Dry-run transfer preview
        uses: EODC-GmbH/github-actions/.github/actions/rclone-move@main
        with:
          source_path: ${{ github.workspace }}/data
          dest_path: remote:bucket/prefix
          include_pattern: '**/*.csv,**/*.parquet'
          dry_run: 'true'

```

Move to a local path on the runner:

```yaml
name: Move To Local Disk
on:
  workflow_dispatch:

jobs:
  movelocal:
    runs-on: [self-hosted, Linux, X64]
    steps:
      - uses: actions/checkout@v4
      - name: Move to local disk
        uses: EODC-GmbH/github-actions/.github/actions/rclone-move@main
        with:
          source_path: ${{ github.workspace }}/cache
          dest_path: /mnt/data/cache
          delete_empty_src_dirs: 'true'
```

## Behavior and defaults

- `size_only: true` uses only file size to decide if a transfer is needed. Set to `false` to allow rclone’s default mtime-based decisions.
- `delete_empty_src_dirs: true` prunes empty directories at the source after moving.
- `dry_run: true` shows what would be moved without transferring data.
- The action echoes the `rclone move` command with resolved flags for visibility.

### Nested paths safety

- When the destination is inside the source (e.g., `src=/parent` and `dst=/parent/subdir`), the action automatically adds an exclude rule for the destination subtree to prevent recursion.
- When the source is inside the destination (e.g., `src=/parent/.stage` and `dst=/parent`), no special handling is required; the move behaves as expected. Your include/exclude patterns are applied relative to `source_path`.

## Troubleshooting

- Patterns not matching: ensure they are quoted in YAML and use rclone filter semantics. Use a dry run to inspect matches.
- Invalid inputs: `transfers` must be a positive integer; `retries` must be a non-negative integer.
- Missing rclone: on non-Ubuntu runners, preinstall rclone or run within a container that has rclone available.
- Authentication: if using a remote (e.g., `s3:`), provide a valid `rclone_config` or ensure the runner has a working config in `$HOME/.config/rclone/rclone.conf`.

## License

This action is distributed under the terms of the repository’s LICENSE.
