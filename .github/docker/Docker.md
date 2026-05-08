# Dockerfile.ci — Walkthrough

This image bakes the full toolchain (Node, Bun, Playwright, Chromium, fonts, etc.) plus all `bun install` dependencies into a single Docker image. CI pulls this instead of installing everything from scratch on every run. Rebuilt weekly, or when `Dockerfile.ci` / `package.json` / `bun.lock` changes.

## Section-by-section

### Lines 3–5 — Base image
Ubuntu 24.04 with `DEBIAN_FRONTEND=noninteractive` so apt never prompts during install.

### Lines 18–21 — Hetzner mirror swap
Ubicloud's runners live in Hetzner's datacenter and observed 90+ second timeouts to `archive.ubuntu.com`. This rewrites apt sources to use Hetzner's local mirror, which is route-local. Uses HTTP because the base image has no `ca-certificates` yet, and apt verifies via GPG signatures, not TLS.

### Lines 26–27 — Apt resilience config
Tells apt to retry 5× per package with 30s timeouts. A single blip won't kill the whole build.

### Lines 31–36 — Core system deps
Installs `git`, `curl`, `unzip`, `xz-utils` (for the Node tarball below), `ca-certificates`, `jq`, `bc`, `gpg`. Wrapped in a 3× retry loop with `sleep 10`.

### Lines 39–47 — GitHub CLI (`gh`)
Adds the GitHub CLI apt repo via signed keyring, then installs `gh`.

### Lines 57–62 — Node.js 22 LTS
Downloads the tarball directly from `nodejs.org` instead of using NodeSource's `setup_22.x` script. Reason: NodeSource's script does its own `apt-get update` + installs `gnupg`, both of which fail on Ubicloud (and `gnupg` was renamed to `gpg` on 24.04 anyway). Direct tarball = one host, no apt.

### Lines 65–67 — Bun
Pinned to v1.3.10. `BUN_INSTALL=/usr/local` so non-root users can run it.

### Line 70 — Claude CLI
Globally installs `@anthropic-ai/claude-code`.

### Line 73 — Playwright system deps
Installs Chromium's runtime libraries (libnss, libatk, etc.) via Playwright's installer.

### Lines 85–90 — Fonts + Xvfb
- **fonts-liberation**: Liberation Sans is a metric-compatible Arial clone. Without it, `make-pdf`'s print CSS falls back to DejaVu Sans and PDFs render incorrectly. Playwright pulls this in implicitly today, but it's installed explicitly to prevent silent regression.
- **xvfb + x11-utils**: virtual display so headed-mode browse tests (`headed-xvfb`, `headed-orphan-cleanup`) can run in CI.

### Lines 97–99 — Pre-install deps
Copies `package.json` AND `bun.lock` (both, for determinism — without the lockfile, transitive deps resolved differently in CI vs local on v1.28.0.0). Runs `bun install --frozen-lockfile`.

### Lines 102–104 — Playwright Chromium
Installs Chromium to `/opt/playwright-browsers` (shared, world-readable) instead of the per-user default, so any UID can use it at runtime.

### Lines 107–110 — Smoke test
Asserts every tool works, including a `fc-match` check that fails the build loudly if Liberation Sans isn't installed (catches the silent PDF regression).

### Lines 115–116 — node_modules cache trick
At runtime, GitHub Actions checks out the repo into `/workspace`, clobbering `/workspace/node_modules`. The build moves it to `/opt/node_modules_cache` and snapshots `package.json` alongside. The CI workflow symlinks it back and validates the snapshot before reusing the cache.

### Lines 121–125 — Non-root runner user
Claude CLI refuses `--dangerously-skip-permissions` when running as root, so a `runner` user is created. `chmod 1777 /tmp` gives a sticky world-writable temp dir; `~/.gstack` and `~/.bun` are pre-created with correct ownership.

## How & why this file helps

`Dockerfile.ci` is the single source of truth for the CI eval environment. Without it, every CI run would install Node, Bun, Playwright, Chromium, and all dependencies from scratch — adding 3–5 minutes per run and exposing builds to network flakiness on every step.

### Why it exists

- **Speed.** Pre-baking the toolchain + `node_modules` cuts CI runs from ~6 min of setup to ~10 sec of pulling a cached image. Across hundreds of runs/week, that's hours saved.
- **Reproducibility.** Versions are pinned (Node 22.20.0, Bun 1.3.10, Ubuntu 24.04). Two CI runs a week apart use identical binaries, so a green build today and a red build tomorrow can't be blamed on a silent toolchain upgrade.
- **Reliability on Ubicloud.** Ubicloud runners live in Hetzner's datacenter and have documented network issues reaching `archive.ubuntu.com` and NodeSource. This image works around both (Hetzner mirror for apt, direct `nodejs.org` tarball for Node) so CI doesn't randomly fail on infrastructure neither side controls.
- **Cost.** Eval runs cost ~$4 each. A flaky setup step that nukes the run halfway through wastes real money. Network-paranoid retries throughout the file are insurance against that.
- **Determinism between local and CI.** Copying `bun.lock` into the image (alongside `package.json`) ensures `bun install --frozen-lockfile` resolves the exact same transitive deps as a developer's laptop. A regression on v1.28.0.0 — where missing the lockfile caused `socks` to land in CI's `node_modules` but not its peers — is the reason this is explicit.

### How it's used

- **Built by** `.github/workflows/ci-image.yml` on a weekly schedule, on changes to this Dockerfile, or on changes to `package.json` / `bun.lock`.
- **Pulled by** the eval workflow (`evals.yml`) on every CI run that needs a real environment (E2E tests, Playwright, Claude CLI invocations).
- **Runtime contract:** GitHub Actions overlays the checked-out repo onto `/workspace`, which would clobber the pre-installed `node_modules`. The image stashes them at `/opt/node_modules_cache` with a `package.json` snapshot; the workflow symlinks them back and validates the snapshot matches before reusing.

### What you'd lose without it

Cold-installing the toolchain on every CI run, fighting Ubicloud's flaky upstream mirrors with no retries, and accepting that any silent dep-resolution drift between local and CI will only surface as red builds at the worst possible moment.



## Key design decisions

1. **Network-paranoid throughout** — every `curl` has `--retry 5 --retry-connrefused`, every apt op wrapped in a 3× retry loop. CI runs cost ~$4/eval; a network blip shouldn't waste that.
2. **Pinned versions** — Node 22.20.0, Bun 1.3.10. Reproducible builds.
3. **Cache-friendly layering** — `package.json` + `bun.lock` are copied as late as possible so toolchain layers stay cached when only deps change.
4. **Defense against silent regressions** — the `fc-match` check fails loudly rather than letting PDFs render wrong.
