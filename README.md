# deb-repo-gen

Generate a Debian/Ubuntu APT repository from `.deb` packages — no external Python dependencies required.

## Features

- Recursively scans one or more input directories for `.deb` files
- Extracts package metadata via `dpkg-deb`
- Multi-distribution, multi-component, multi-architecture support
- Full APT metadata: `Packages`, `Packages.gz`, `Release`, `InRelease`, `Release.gpg`
- `by-hash/` support — hash-named index copies for atomic APT updates
- `Contents-<arch>.gz` indices (`--contents`) for `apt-file` lookups
- Optional GPG signing
- Adds packages to **existing** repositories — auto-discovers pool content
- Incremental updates — only processes changed packages
- Idempotent — repeated runs produce identical output
- Duplicate and conflict detection
- Corrupted `.deb` files are skipped and logged, never abort
- Resumable after interruption thanks to in-progress state markers
- Built-in repository consistency validation
- Dry-run and verbose modes
- Falls back to pure-Python generation when `apt-ftparchive` is unavailable

## Requirements

| Dependency         | Required? | Purpose                                      |
|-------------------|-----------|----------------------------------------------|
| Python 3.8+       | Required  | Runtime                                      |
| `dpkg-deb`        | Required  | `.deb` metadata extraction                   |
| `apt-ftparchive`  | Optional  | Faster metadata generation (`apt-utils`)     |
| `gpg`             | Optional  | Signing Release files (`--sign-key`)         |

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
deb-repo-gen -i <input-dir> -o <output-dir> --dists <codenames> [options]
```

### Required arguments

| Flag               | Description                                                  |
|--------------------|--------------------------------------------------------------|
| `-i`, `--input`    | Input directory with `.deb` files (repeatable)               |
| `-o`, `--output`   | Output directory for the APT repository                      |
| `--dists`          | Comma-separated distribution codenames (e.g. `jammy,noble`)  |

### Optional arguments

| Flag                     | Description                                                  |
|--------------------------|--------------------------------------------------------------|
| `--archs`                | Comma-separated architectures (default: `amd64`)             |
| `--components`           | Comma-separated components (default: `main`)                 |
| `--default-component`    | Fallback component (default: `main`)                         |
| `--section-map`          | JSON mapping Section prefixes → components                   |
| `--origin`               | `Origin` field in Release files (default: `Custom Repository`)|
| `--label`                | `Label` field in Release files (default: `Custom APT Repository`)|
| `--description`          | `Description` field in Release files                         |
| `--contents`             | Generate `Contents-<arch>.gz` file-to-package indices        |
| `--sign-key`             | GPG key ID/email/fingerprint for signing                     |
| `-n`, `--dry-run`        | Show what would be done without making changes               |
| `-f`, `--force`          | Force full regeneration (ignore incremental state)           |
| `--no-apt-ftparchive`    | Use pure-Python fallback instead of `apt-ftparchive`         |
| `--version`              | Print version and exit                                       |
| `-v`, `--verbose`        | Verbose output                                               |
| `--debug`                | Debug output                                                 |
| `-q`, `--quiet`          | Suppress non-error output                                    |

## Examples

```bash
# Basic repository with a single distribution and architecture
deb-repo-gen -i /path/to/debs -o /srv/apt-repo --dists jammy --archs amd64

# Multiple input directories and distributions
deb-repo-gen -i /debs/main -i /debs/extra -o /srv/apt-repo \
    --dists jammy,noble --components main,universe

# Add packages to an existing repository (auto-discovers pool content)
deb-repo-gen -i /new/debs -o /srv/existing-repo --dists jammy --archs amd64

# Custom Release metadata
deb-repo-gen -i /debs -o /srv/apt-repo --dists jammy \
    --origin "MyCompany" --label "Internal APT" \
    --description "Internal packages for jammy"

# With Contents indices (enables apt-file)
deb-repo-gen -i /debs -o /srv/apt-repo --dists jammy \
    --archs amd64 --contents

# With GPG signing
deb-repo-gen -i /debs -o /srv/apt-repo --dists jammy \
    --archs amd64 --sign-key DEADBEEF

# Dry-run to preview changes
deb-repo-gen -i /debs -o /srv/apt-repo --dists jammy --dry-run --verbose

# Force full regeneration
deb-repo-gen -i /debs -o /srv/apt-repo --dists jammy --force

# Without apt-ftparchive (pure Python)
deb-repo-gen -i /debs -o /srv/apt-repo --dists jammy --no-apt-ftparchive

# Section-to-component mapping (non-free → restricted, contrib → multiverse)
deb-repo-gen -i /debs -o /srv/apt-repo --dists jammy \
    --section-map '{"non-free/": "restricted", "contrib/": "multiverse"}'
```

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
| 2    | Input directory not found   |
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

## Limitations

- **Single-instance only** — running two instances against the same output
  directory concurrently will corrupt the incremental state and produce
  inconsistent metadata. The in-progress marker detects interrupted runs but
  does not actively lock the repository.
- **`dpkg-deb` required on every run** — even for existing pool packages
  during discovery, metadata is re-extracted. On very large repositories
  this adds noticeable startup time.
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
- **`_KNOWN_ARCHITECTURES` unused** — the frozenset of known Debian
  architecture names is declared in the source but never validated against
  user input. Any architecture string is accepted.
- **No progress bar** — large repositories may take considerable time with
  no visual feedback beyond per-file log messages (visible with `--verbose`).
- **No built-in HTTP server** — the output directory must be served by an
  external web server (nginx, Apache, Python `http.server`, etc.).

## License

MIT
