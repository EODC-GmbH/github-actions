# rclone-fake-move (GitHub Action)

Emulates `rclone move` safely when the source is inside the destination. This avoids rclone’s "overlapping remotes" error while preserving move semantics without deleting files created after the copy begins.

## Overview

- Installs `rclone` if missing
- Accepts inline rclone config (pass `${{ secrets.RCLONE_CONFIG }}`)
- Applies include/exclude via a generated `--filter-from` file
- Enumerates files once, then copy+delete that exact list
- Optionally prunes empty source directories

## When to use this action

Use this action if:

- `source_path` is a subdirectory of `dest_path` (e.g., staging into its parent), and
- You want move semantics, but rclone refuses with overlapping-paths error.

If your destination is inside the source subtree, you can also use the standard `rclone-move` action (it auto-excludes the destination subtree and runs `rclone move`).

## Inputs

Same as `rclone-move`:

- `source_path`, `dest_path`, `transfers`, `retries`, `size_only`, `dry_run`, `delete_empty_src_dirs`, `include_pattern`, `exclude_pattern`, `rclone_config`.

## Behavior

1) Build filter file (includes `+ /**/`, user excludes then includes, `- *` if includes exist). If destination is inside source, auto-exclude the destination subtree to prevent recursion.

2) Enumerate files once:

```
rclone lsf -R --files-only --filter-from <filter> <source>
```

3) Copy exactly that list, then delete exactly that list:

```
rclone copy --files-from-raw <list> <source> <dest>
rclone delete --files-from-raw <list> <source>
```

4) Optionally prune empty dirs:

```
rclone rmdirs --leave-root <source>
```

This avoids deleting files that appear after the copy began.

## Example

```yaml
name: Move cams_ads Data (safe overlap)
on:
  workflow_dispatch:

jobs:
  move:
    runs-on: [self-hosted, linux, cams_ads]
    steps:
      - uses: actions/checkout@v4
      - name: Move into datastore (safe overlap)
        uses: EODC-GmbH/github-actions/.github/actions/rclone-fake-move@v2
        with:
          source_path: /eodc/products/cams_ads/.stage
          dest_path: /eodc/products/cams_ads/
          include_pattern: '*.nc'
          exclude_pattern: '**/*.tmp,**/.stage/**'
          transfers: '8'
          retries: '5'
          size_only: 'true'
          delete_empty_src_dirs: 'true'
          dry_run: 'true'
          rclone_config: ${{ secrets.RCLONE_CONFIG }}
```

## Notes

- Dry runs show both copy and delete operations without changing data.
- If no files match, the action exits successfully with no operations.
- For non-overlapping paths, prefer `rclone-move` which uses native `rclone move`.

