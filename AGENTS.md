# AGENTS.md

## Architecture

Single-file Python CLI (`deb-repo-gen`, ~2700 lines) with **zero external Python dependencies** (stdlib only). Generates APT repositories from `.deb` packages.

## Quirks

- **No tests, no lint/formatter/typecheck config.** The only way to verify changes is running the tool against real `.deb` files.
- **`--quick` mode forces pure-Python generation** — it overrides `use_apt` to `False` even if `apt-ftparchive` is installed, because apt-ftparchive can't do incremental patching of Packages files.
- **`--trust-pool` reads package metadata from state.json and existing Packages files** without running `dpkg-deb` on existing pool `.deb`s. This is what `--quick` implies.
- **Atomic file writes use temp files in the output directory** (not `/tmp`), created alongside the target. Ensure the output filesystem has space for both old and new file versions during replacement.
- **Single-instance only** — concurrent runs against the same output directory will corrupt state.
- **State file** lives at `<output>/.repo-state/state.json` (schema version 2). An `in_progress.json` marker is used for resumability after interruption.
- **Python 3.8+** is the minimum version due to `from __future__ import annotations`.
- **File permissions** — atomic writes set mode `0o644`.

## GPG signing flow

- **`sign_key` is persisted in state.json.** Once you run with `--sign-key`, subsequent runs auto-detect it — you never need to specify the key again.
- **`--no-sign`** skips all GPG operations (neither sign nor remove stale sigs). Existing `InRelease`/`Release.gpg` are left as-is.
- **`--sign-only`** skips everything except Release regeneration + signing. Auto-detects `--dists`/`--components`/`--archs` from the existing `dists/` directory. Combine with `--generate-key` for first-time key creation on an existing unsigned repo.
- **`--generate-key --key-email X`** creates an RSA 4096 key with no expiry and no passphrase. Skips generation if a key for that email already exists. Requires `--key-email`.
- **On no-op runs** (`to_add` and `to_remove` both empty, no `--force`), Release generation and signing are skipped entirely.
- **Mutually exclusive**: `--sign-only` / `--no-sign`, `--sign-key` / `--no-sign`, `--generate-key` / `--no-sign`.
- **Auto-detection in `__init__`** loads state early to resolve the signing key before the pipeline starts. The `sign_only` branch then overrides dists/components/archs via `_auto_detect_repo_structure()`.

## Required system tools

| Tool | Package | Purpose |
|------|---------|---------|
| `dpkg-deb` | `dpkg-dev` | Required. Extracts `.deb` metadata. |
| `apt-ftparchive` | `apt-utils` | Optional. Faster metadata generation. |
| `gpg` | `gnupg` | Optional. Signing Release files. Required by `--sign-key`, `--sign-only`, `--generate-key`. |

## Exit codes

0=success, 1=invalid args, 2=input not found, 3=no .deb found, 4=missing dependency, 5=corrupted package, 6=duplicate/conflict, 7=GPG error, 8=validation failed, 9=interrupted, 10=disk error, 99=unknown.
