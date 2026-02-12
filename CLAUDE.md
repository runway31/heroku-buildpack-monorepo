# CLAUDE.md

## Project Overview

This is the **Heroku Monorepo Buildpack** — a Heroku buildpack that enables deploying multiple applications from a single monorepo. It copies a specified subdirectory (set via the `APP_BASE` environment variable) to the build root so that subsequent buildpacks treat it as the application.

Based on [heroku-buildpack-multi-procfile](https://github.com/heroku/heroku-buildpack-multi-procfile), but copies the entire app directory to root rather than just the Procfile.

## Repository Structure

```
bin/
  compile    # Main build step — copies APP_BASE subdirectory to build root
  detect     # Buildpack detection — always matches (echoes "Monorepo", exits 0)
  release    # Release config — outputs empty YAML (--- {})
README.md    # Usage documentation
```

This is a minimal, pure-Bash project with no external dependencies, no package manager, and no build tooling.

## How the Buildpack Works

Heroku buildpacks implement three scripts: `detect`, `compile`, and `release`.

### bin/detect
Always returns success. Any app can use this buildpack.

### bin/compile
Receives three arguments from Heroku: `BUILD_DIR`, `CACHE_DIR`, `ENV_DIR`.

1. Reads `APP_BASE` from `${ENV_DIR}/APP_BASE` (fails if not set)
2. Iterates through items in `${BUILD_DIR}/${APP_BASE}/` and removes symlinks that conflict with real directories at the build root (enables local dev symlinks like `engines -> ../../engines` while using the real root directory on Heroku)
3. Recursively copies `${BUILD_DIR}/${APP_BASE}/.` into `${BUILD_DIR}`
4. Verifies the copy succeeded (exits 1 on failure)
5. Lists the final build directory contents

### bin/release
Outputs `--- {}` — no additional release-phase processes.

## Code Conventions

- **Language**: Bash (`#!/usr/bin/env bash`)
- **Output formatting**: All user-facing output is piped through `indent()` which adds 6-space indentation via `sed -u 's/^/      /'` (standard Heroku buildpack convention)
- **Error handling**: Check exit codes with `$? -ne 0` and `exit 1` on failure
- **Variable quoting**: Always quote variables with `"${VAR}"` to handle paths with spaces

## Configuration

The buildpack uses a single environment variable:

- **`APP_BASE`** (required): Relative path from the repo root to the application directory. Set via `heroku config:set APP_BASE=path/to/app`.

When using multiple buildpacks, this buildpack must be added first (`-i 1`).

## Testing

There is no automated test suite. The buildpack is validated manually by deploying to Heroku and verifying the correct subdirectory is copied to the build root.

## Common Tasks

### Modifying build behavior
Edit `bin/compile`. The script is ~40 lines of Bash.

### Changing detection logic
Edit `bin/detect`. Currently unconditional (always matches).

### Adding release-phase configuration
Edit `bin/release`. Currently outputs empty YAML.

## Key Design Decisions

- **Symlink removal before copy**: Symlinks in the app subdirectory that point to directories already existing at root are removed before copying. This prevents symlinks (used for local development convenience) from overwriting real directories during the Heroku build.
- **Copy instead of move**: Uses `cp -r` rather than `mv` because the build directory filesystem may be read-only for move operations, and the source files at root level may still be needed by subsequent buildpacks.
- **Always-match detection**: `bin/detect` always succeeds because the buildpack is explicitly added by users who need monorepo support — there's no file-based heuristic to detect.
