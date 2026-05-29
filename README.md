# deb-repo-gen

Generate a Debian/Ubuntu APT repository from `.deb` packages — no external Python dependencies required.

## Features

- Recursively scans one or more input directories for `.deb` files, or accepts individual `.deb` files directly
- Extracts package metadata via `dpkg-deb`
- Multi-distribution, multi-component, multi-architecture support
- Full APT metadata: `Packages`, `Packages.gz`, `Release`, `InRelease`, `Release.gpg`
- `by-hash/` support — hash-named index copies for atomic APT updates
- `Contents-<arch>.gz` indices (`--contents`) for `apt-file` lookups
- Optional GPG signing with **key persistence** — specify the key once, it's auto-detected on subsequent runs
- **Sign-only mode** — (re)sign an existing repository without re-processing packages
- **Auto key generation** — create RSA 4096 keys directly from the tool (`--generate-key`)
- Adds packages to **existing** repositories — auto-discovers pool content
- **Quick mode** — fast incremental addition of packages to large repos
- **Incremental updates** — only processes changed packages
- **Incremental Packages patching** — modifies existing index files instead of regenerating from scratch
- Idempotent — repeated runs produce identical output
- Duplicate and conflict detection
- Corrupted `.deb` files are skipped and logged, never abort
- Resumable after interruption thanks to in-progress state markers
- Built-in repository consistency validation
- Progress bars for scanning, pool discovery, and metadata generation
- Dry-run and verbose modes
- Falls back to pure-Python generation when `apt-ftparchive` is unavailable

## Requirements

| Dependency         | Required? | Purpose                                      |
|-------------------|-----------|----------------------------------------------|
| Python 3.8+       | Required  | Runtime                                      |
| `dpkg-deb`        | Required  | `.deb` metadata extraction                   |
| `apt-ftparchive`  | Optional  | Faster metadata generation (`apt-utils`)     |
| `gpg`             | Optional  | Signing Release files. Required by `--sign-key`, `--sign-only`, `--generate-key`. |

The script uses **only the Python standard library** — no pip packages needed.

## Install

```bash
# Make executable and place on PATH
chmod +x deb-repo-gen
sudo cp deb-repo-gen /usr/local/bin/

# Ensure required tools are installed
sudo apt install dpkg-dev              # provides dpkg-deb
sudo apt install apt-utils gpg         # optional but recommended
```

## Usage

```
deb-repo-gen -i <path> -o <output-dir> --dists <codenames> [options]
```

`-i` accepts directories (scanned recursively for `.deb` files) **or** individual `.deb` files.

### Required arguments

| Flag               | Description                                                  |
|--------------------|--------------------------------------------------------------|
| `-i`, `--input`    | Input directory or `.deb` file (repeatable). Not required with `--sign-only`. |
| `-o`, `--output`   | Output directory for the APT repository                      |
| `--dists`          | Comma-separated distribution codenames (e.g. `jammy,noble`). Not required with `--sign-only` (auto-detected). |

### Optional arguments

| Flag                     | Description                                                  |
|--------------------------|--------------------------------------------------------------|
| `--archs`                | Comma-separated architectures (default: `amd64`)             |
| `--components`           | Comma-separated components (default: `main`)                 |
| `--section-map`          | JSON mapping Section prefixes → components                   |
| `--contents`             | Generate `Contents-<arch>.gz` file-to-package indices        |
| `--sign-key`             | GPG key ID/email/fingerprint for signing. Persisted to state — only needed once. |
| `--sign-only`            | Only (re)sign an existing repo — skips scanning, pool population, and validation |
| `--no-sign`              | Skip all GPG operations entirely — leave existing signatures untouched |
| `--generate-key`         | Generate an RSA 4096 GPG key pair for signing (no passphrase). Requires `--key-email`. |
| `--key-name`             | Real name for auto-generated GPG key (default: `APT Repository`) |
| `--key-email`            | Email for auto-generated GPG key (required with `--generate-key`) |
| `-n`, `--dry-run`        | Show what would be done without making changes               |
| `-f`, `--force`          | Force full regeneration (ignore incremental state)           |
| `--trust-pool`           | Trust existing pool — skip re-analysis of destination repo   |
| `--skip-validation`      | Skip repository consistency validation after generation      |
| `--quick`                | Fast incremental add — implies `--trust-pool` and            |
|                          | `--skip-validation`, patches Packages instead of regen       |
| `--no-apt-ftparchive`    | Use pure-Python fallback instead of `apt-ftparchive`         |
| `-v`, `--verbose`        | Verbose output                                               |
| `--debug`                | Debug output                                                 |
| `-q`, `--quiet`          | Suppress non-error output                                    |
| `--version`              | Print version and exit                                       |

## Examples

### Quickest — add a single package to an existing repo

```bash
deb-repo-gen -i nginx_1.24.0-1_amd64.deb -o /srv/apt-repo --dists jammy --quick
```

`--quick` patches the existing Packages files incrementally (no full rescan, no
re-validation). Ideal for CI/CD pipelines pushing one or a few packages at a
time. Metadata (Packages, Release, signatures) is always refreshed.

### Quick — add a few packages with a dry-run first

```bash
# Preview what would happen
deb-repo-gen -i /new-debs/ -o /srv/apt-repo --dists jammy --quick --dry-run -v

# Then actually run it
deb-repo-gen -i /new-debs/ -o /srv/apt-repo --dists jammy --quick
```

### Moderate — standard incremental update

```bash
deb-repo-gen -i /new-debs/ -o /srv/apt-repo --dists jammy --archs amd64
```

Scans the input directory, discovers new/changed packages, and only
regenerates affected components. Existing pool packages are discovered
via existing Packages indices when available (fast), falling back to
`dpkg-deb` extraction for any unindexed packages.

### Moderate — incremental with GPG signing

```bash
deb-repo-gen -i /new-debs/ -o /srv/apt-repo --dists jammy \
    --archs amd64 --sign-key admin@example.com
```

### Moderate — trust the existing pool, skip validation

```bash
deb-repo-gen -i /new-debs/ -o /srv/apt-repo --dists jammy \
    --archs amd64 --trust-pool --skip-validation
```

`--trust-pool` loads existing package metadata from state and Packages
files (no `dpkg-deb` calls for existing packages). `--skip-validation`
avoids the post-generation checksum verification pass. Unlike `--quick`,
this still does full Packages regeneration and stale metadata cleanup.

### GPG signing — first time setup

```bash
# Generate a key and sign an existing repo in one step
deb-repo-gen --sign-only --generate-key --key-email admin@example.com \
    -o /srv/apt-repo

# Or create a fresh signed repo from scratch
deb-repo-gen -i /all-debs/ -o /srv/apt-repo --dists jammy \
    --archs amd64 --sign-key admin@example.com
```

The signing key is automatically persisted to `.repo-state/state.json`. On all
subsequent runs, the tool reuses it — you never need to specify `--sign-key`
again.

### GPG signing — subsequent runs (key auto-detected)

```bash
# Add packages — key is reused automatically from state
deb-repo-gen -i /new-debs/ -o /srv/apt-repo --dists jammy --quick
```

### GPG signing — add packages without touching signatures

```bash
# Update metadata but leave existing InRelease/Release.gpg alone
deb-repo-gen -i /new-debs/ -o /srv/apt-repo --dists jammy --no-sign
```

Useful in CI when signing is handled by a separate step. Run `deb-repo-gen --sign-only -o /srv/apt-repo`
afterwards to re-sign.

### GPG — distribute the public key to clients

```bash
# Export the public key
gpg --export --armor admin@example.com > my-repo-keyring.asc

# On each client, install it
sudo cp my-repo-keyring.asc /usr/share/keyrings/my-repo-keyring.asc
```

Then in `/etc/apt/sources.list.d/custom.list`:

```
deb [signed-by=/usr/share/keyrings/my-repo-keyring.asc] https://server/apt/ jammy main
```

### Full — first-time repository creation

```bash
deb-repo-gen -i /all-debs/ -o /srv/apt-repo \
    --dists jammy,noble --archs amd64,arm64 \
    --components main,universe
```

Full scan, full generation, full validation. This is the slowest but
most thorough mode.

### Full — with Contents indices and signing

```bash
deb-repo-gen -i /all-debs/ -o /srv/apt-repo --dists jammy \
    --archs amd64,arm64 --contents --sign-key DEADBEEF
```

`--contents` generates `Contents-<arch>.gz` indices (enables `apt-file`
lookups) by running `dpkg-deb -c` on every package. This is
significantly slower — avoid unless needed.

### Full — section mapping

```bash
deb-repo-gen -i /all-debs/ -o /srv/apt-repo --dists jammy \
    --section-map '{"non-free/": "restricted", "contrib/": "multiverse"}'
```

### Full — force regeneration of everything

```bash
deb-repo-gen -i /all-debs/ -o /srv/apt-repo --dists jammy --force
```

Ignores incremental state and regenerates all metadata from scratch.

## Disk space considerations

Atomic file writes use temporary files created **alongside the target**
(in the output directory, not in `/tmp`). This means:

- **The output filesystem must have enough space for both the old and new
  version of each file** during replacement. For large Packages or Release
  files this is negligible, but for `.deb` copies the temporary file is as
  large as the package itself.
- **Pool growth is cumulative** — packages are never pruned automatically.
  A repository holding 1,000 packages averaging 50 MB each consumes ~50 GB
  in `pool/` alone, plus metadata overhead.
- **`by-hash/` stores up to 3 extra copies** of each Packages/Packages.gz
  file (one per hash algorithm). On repos with many distributions,
  components, and architectures, this can add up.
- **`--contents` doubles I/O** — `Contents-<arch>.gz` files can be large
  (hundreds of MB for repos with tens of thousands of packages) and are
  written atomically, requiring transient extra disk space.
- **Monitor disk usage** with `du -sh /srv/apt-repo/pool/` and
  `du -sh /srv/apt-repo/dists/` to catch growth before it becomes a
  problem. The `.repo-state/state.json` file also grows with the number
  of tracked packages.

## Repository layout

```
<output>/
├── pool/
│   └── <component>/
│       └── <prefix>/
│           └── <package>/
│               └── <package>_<version>_<arch>.deb
├── dists/
│   └── <dist>/
│       ├── Release
│       ├── InRelease              (if signed)
│       ├── Release.gpg            (if signed)
│       ├── Contents-<arch>.gz     (if --contents)
│       └── <component>/
│           └── binary-<arch>/
│               ├── Packages
│               ├── Packages.gz
│               └── by-hash/
│                   ├── MD5Sum/<digest>
│                   ├── SHA1/<digest>
│                   └── SHA256/<digest>
└── .repo-state/
    └── state.json                 (incremental state)
```

## Serving the repository

Serve the output directory with any HTTP server:

```bash
# Quick test with Python
python3 -m http.server 8080 -d /srv/apt-repo

# Production with nginx
# location / { root /srv/apt-repo; autoindex on; }
```

Clients add it to their sources:

```bash
echo "deb [trusted=yes] http://<host>:<port>/ <dist> <components>" \
    > /etc/apt/sources.list.d/custom.list
```

If signed, replace `[trusted=yes]` with `signed-by=/etc/apt/keyrings/custom.asc`.

## Exit codes

| Code | Meaning                     |
|------|-----------------------------|
| 0    | Success                     |
| 1    | Invalid arguments           |
| 2    | Input path not found        |
| 3    | No `.deb` files found       |
| 4    | Missing dependency          |
| 5    | Corrupted package           |
| 6    | Duplicate/conflict detected |
| 7    | GPG signing failed          |
| 8    | Repository validation failed|
| 9    | Interrupted (SIGINT/SIGTERM)|
| 10   | Disk/OS error               |
| 99   | Unknown error               |

## Incremental operation

On first run, the tool performs a full generation and saves state to
`<output>/.repo-state/state.json`. Subsequent runs compare scanned packages
against this state and only process new, changed, or removed packages.

Use `--force` to ignore saved state and regenerate everything.

### Quick mode

`--quick` is optimized for the common workflow of adding a few packages to an
existing large repository. It:

- Implies `--trust-pool` — loads existing packages from state/Packages files
  instead of re-running `dpkg-deb` on every `.deb` in the pool
- Implies `--skip-validation` — skips the post-generation checksum pass
- Patches existing `Packages` files incrementally (only adds/replaces/removes
  changed stanzas) instead of regenerating from scratch
- Forces pure-Python mode (`apt-ftparchive` does not support patching)
- Skips stale metadata cleanup

Metadata (Packages, Packages.gz, Release, by-hash, signatures) is **always
refreshed** — only the *method* changes (patch vs. full regen).

## Limitations

- **Single-instance only** — running two instances against the same output
  directory concurrently will corrupt the incremental state and produce
  inconsistent metadata. The in-progress marker detects interrupted runs but
  does not actively lock the repository.
- **No version pruning** — all versions of a package present in the input
  directories or pool are retained indefinitely. There is no `--keep-latest`
  or automatic old-version cleanup.
- **No source package (`.dsc`) support** — only binary `.deb` packages are
  handled. Source repositories (with `Sources.gz` indices) are not generated.
- **No `udeb` support** — Debian installer micro-packages are not recognised.
- **No i18n / Translation indices** — `Translation-en` files for translated
  package descriptions are not generated.
- **No DEP-11 / AppStream metadata** — YAML-based AppStream component metadata
  is not generated.
- **Manual vs apt-ftparchive discrepancy** — in `--no-apt-ftparchive` mode,
  the `Packages` file is built exclusively from the in-memory package list
  (input scan + pool discovery). In apt mode, `apt-ftparchive` scans the
  `pool/` directory directly. If a `.deb` file is manually dropped into
  `pool/` between runs, it will appear in apt mode but be silently missed
  in manual mode until the next pool discovery pass.
- **`Contents` overhead** — generating `Contents-<arch>.gz` requires running
  `dpkg-deb -c` on every package, which is significantly slower than basic
  metadata extraction. Enable with `--contents` only when needed.
- **Validation checksums ignore `Release` self-reference** — apt-ftparchive
  sometimes produces stale self-referencing checksum entries. These are
  deliberately skipped during validation.

## License

MIT
