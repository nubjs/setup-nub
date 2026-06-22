# setup-nub

Install the [nub](https://github.com/nubjs/nub) CLI on a GitHub Actions runner. A **drop-in replacement for [`actions/setup-node`](https://github.com/actions/setup-node)** — swap the `uses:` line and your existing workflow keeps working.

```yaml
- uses: actions/checkout@v4   # must run before setup-nub — it reads the project's pin files
- uses: nubjs/setup-nub@v0
- run: nub install
- run: nub run build
```

That's the whole story for most projects: setup-nub puts `nub` on PATH, **eagerly provisions the project's pinned Node** (resolved from `.node-version` / `.nvmrc` / `package.json`) so the first build step is warm instead of paying a Node download mid-build, **caches nub's store across runs by default**, and reads a standard `.npmrc` for registry auth.

Because the eager provision reads the project's pin files off disk, **`actions/checkout` must run before `setup-nub`.** With no inputs and no pin declared, the action provisions nothing and nub falls back to provisioning lazily at runtime — it never fails on a missing pin.

## Drop-in from setup-node

`setup-nub` accepts every `actions/setup-node` input and never errors on a recognized one. Migrate by changing one line:

```yaml
# before
- uses: actions/setup-node@v4
  with:
    node-version: 20
    cache: npm
    registry-url: https://registry.npmjs.org

# after
- uses: nubjs/setup-nub@v0
  with:
    node-version: 20          # pre-provisions Node 20 into nub's cache (warm-up hint)
    cache: npm                # accepted; caching is on by default regardless of value
    registry-url: https://registry.npmjs.org
```

Like setup-node, **caching is on by default** — `cache: npm` (or `yarn`/`pnpm`/`bun`) keeps working but is no longer required to get a warm store. Disable with `package-manager-cache: false`.

## How nub's Node provisioning differs from setup-node

`setup-node` exists to **put a Node toolchain on the runner**. nub does that itself, from the project's declared pin. So `node-version` here means something subtly different:

- With no inputs (the common case), setup-nub **eagerly provisions the project's own pinned Node** up front — it resolves the full pin chain (`devEngines.runtime` → `.node-version` → `.nvmrc` → `engines.node`) and installs the concrete version, so later steps don't pay a lazy download. If the project declares no pin, it skips gracefully (no error) and nub provisions lazily at runtime.
- `node-version` and `node-version-file` are **warm-up hints**, not pins. They pre-provision that Node into nub's cache — but **nub still runs the project's declared Node at runtime.** If the input disagrees with the project's resolved pin, the action emits a warning and the project pin wins.

This is the one place the drop-in semantics bend: setup-node's `node-version` overrides; setup-nub's pre-warms.

## Inputs

| Input | Default | Behavior |
|---|---|---|
| `nub-version` | `latest` | Version of nub to install — any npm semver range (`0.0.47`, `^0.0`, `latest`). |
| `node-version` | — | Pre-provision this Node into nub's cache (warm-up hint, not a pin; warns on mismatch). |
| `node-version-file` | — | Read a Node version from this file (`.node-version`, `.nvmrc`, `package.json`) and pre-provision it. |
| `cache` | auto | Explicitly enable/disable caching of nub's global store and provisioned Node toolchains. A **boolean**; a setup-node PM name (`npm`/`yarn`/`pnpm`/`bun`) is also accepted and treated as truthy. Leave **unset** to auto-enable when the project looks installable (mirrors setup-node). |
| `package-manager-cache` | `true` | Set to `false` to disable the automatic caching. Mirrors setup-node's input of the same name. An explicit `cache` value still wins. |
| `cache-dependency-path` | auto-detect | Lockfile path(s) whose hash keys the cache. Globs / newline-delimited lists. |
| `cache-key-prefix` | — | Prefix injected into the cache key, to scope or bust caches independently of the lockfile. |
| `working-directory` | checkout root | Directory to resolve the project's Node pin and lockfile from. For monorepos where `package.json`/`.node-version` live in a subdirectory. |
| `registry-url` | — | Registry to set up for auth. Writes a temporary user-level `.npmrc` (via `NPM_CONFIG_USERCONFIG`) and wires auth to `env.NODE_AUTH_TOKEN`. |
| `scope` | repo owner | Scope for a scoped registry. Falls back to the repo owner for GitHub Packages. |
| `always-auth` | `false` | Write `always-auth=true` into the `.npmrc`. |
| `token` | `github.token` | GitHub-API rate-limit relief when resolving nub's version range. |

Accepted for setup-node compatibility but **ignored** (never errors): `check-latest`, `architecture`, `mirror`, `mirror-token`.

> The `version` input is a **deprecated** alias for `nub-version`, kept for one minor. It emits a warning; use `nub-version`.

## Outputs

| Output | Description |
|---|---|
| `nub-version` | The installed nub version (bare `v<semver>`). |
| `node-version` | The Node version nub resolves for the project. Empty when nothing was provisioned. |
| `cache-hit` | Whether an exact cache match was restored — `true` on an exact key hit, `false` on a `restore-keys` partial hit, empty on a full miss (e.g. the first run), mirroring `actions/cache`. |
| `caching-enabled` | Whether caching is active for this run (`true`/`false`), reflecting the resolved `cache`/`package-manager-cache`/auto-detect decision — independent of whether a cache was hit. |

## Caching

**Caching is on by default**, mirroring `actions/setup-node`. It auto-enables when the project looks installable — a lockfile (`pnpm-lock.yaml`, `package-lock.json`, `bun.lock`/`bun.lockb`, `yarn.lock`) is present, or `package.json` declares a `packageManager` or `devEngines` field. The action caches nub's durable, cross-run directories:

- the global content-addressed store (resolved from `nub store path` — the big win)
- the provisioned Node toolchain dir (`<cache>/node`)
- the PM packument cache (best-effort)

The cache key is `nub-<os>-<arch>-<prefix>-<hash(node-pin)>-<hash(lockfile)>` with a `restore-keys` ladder so a fresh lockfile still gets a warm store, and a Node-version bump invalidates the stale toolchain. `cache-key-prefix` injects the `<prefix>` segment for scoping or busting caches independently of the lockfile.

To **disable** caching, set `package-manager-cache: false` (or `cache: false`). An explicit `cache` value always wins over the auto-detection.

## Registry auth

`registry-url` writes a standard, neutral, temporary user-level `.npmrc` — byte-for-byte the setup-node authutil contract — into `$RUNNER_TEMP/.npmrc` and points npm config at it via `NPM_CONFIG_USERCONFIG`:

```ini
@scope:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

Set the token the same way you would for setup-node:

```yaml
- uses: nubjs/setup-nub@v0
  with:
    registry-url: https://npm.pkg.github.com
    scope: "@my-org"
- run: nub install
  env:
    NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Versioning

- `nubjs/setup-nub@v0` — floating tag, gets fixes (recommended).
- `nubjs/setup-nub@v0.1.0` — pinned to a specific release.

## License

MIT
