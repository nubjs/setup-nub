# setup-nub

Install the [nub](https://github.com/nubjs/nub) CLI on a GitHub Actions runner. A **drop-in replacement for [`actions/setup-node`](https://github.com/actions/setup-node)** — swap the `uses:` line and your existing workflow keeps working.

```yaml
- uses: nubjs/setup-nub@v1
- run: nub install
- run: nub run build
```

That's the whole story for most projects: nub provisions the project's pinned Node (from `.node-version` / `.nvmrc` / `package.json`) the first time it runs, and reads a standard `.npmrc` for registry auth — so the action's job is just to put `nub` on PATH and stay out of the way.

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
- uses: nubjs/setup-nub@v1
  with:
    node-version: 20          # pre-provisions Node 20 into nub's cache (warm-up hint)
    cache: true               # boolean — nub has one store regardless of lockfile
    registry-url: https://registry.npmjs.org
```

## How nub's Node provisioning differs from setup-node

`setup-node` exists to **put a Node toolchain on the runner**. nub does that itself, at runtime, from the project's declared pin. So `node-version` here means something subtly different:

- `node-version` and `node-version-file` are **warm-up hints**, not pins. They pre-provision that Node into nub's cache so the first `nub` call is warm — but **nub still runs the project's declared Node at runtime.** If the input disagrees with the project's resolved pin, the action emits a warning and the project pin wins.
- Leave both empty (the common case) and nub resolves + provisions the pin lazily on first use.

This is the one place the drop-in semantics bend: setup-node's `node-version` overrides; setup-nub's pre-warms.

## Inputs

| Input | Default | Behavior |
|---|---|---|
| `nub-version` | `latest` | Version of nub to install — any npm semver range (`0.0.47`, `^0.0`, `latest`). |
| `node-version` | — | Pre-provision this Node into nub's cache (warm-up hint, not a pin; warns on mismatch). |
| `node-version-file` | — | Read a Node version from this file (`.node-version`, `.nvmrc`, `package.json`) and pre-provision it. |
| `cache` | `false` | Cache nub's global store and provisioned Node toolchains across runs (**boolean**, not a PM name). |
| `cache-dependency-path` | auto-detect | Lockfile path(s) whose hash keys the cache. Globs / newline-delimited lists. |
| `registry-url` | — | Registry to set up for auth. Writes a neutral project `.npmrc` and wires auth to `env.NODE_AUTH_TOKEN`. |
| `scope` | repo owner | Scope for a scoped registry. Falls back to the repo owner for GitHub Packages. |
| `always-auth` | `false` | Write `always-auth=true` into the `.npmrc`. |
| `token` | `github.token` | GitHub-API rate-limit relief when resolving nub's version range. |

Accepted for setup-node compatibility but **ignored in v1** (never errors): `check-latest`, `architecture`, `mirror`, `mirror-token`.

> The `version` input is a **deprecated** alias for `nub-version`, kept for one minor. It emits a warning; use `nub-version`.

## Outputs

| Output | Description |
|---|---|
| `nub-version` | The installed nub version (bare `v<semver>`). |
| `node-version` | The Node version nub resolves for the project. Empty when nothing was provisioned. |
| `cache-hit` | Whether nub's store cache was restored (only meaningful with `cache: true`). |

## Caching

With `cache: true`, the action caches nub's durable, cross-run directories:

- the global content-addressed store (resolved from `nub store path` — the big win)
- the provisioned Node toolchain dir (`<cache>/node`)
- the PM packument cache (best-effort)

The cache key is `nub-<os>-<arch>-<hash(lockfile)>` with a `restore-keys` ladder so a fresh lockfile still gets a warm store. `cache` defaults to `false` for v1 (opt-in).

## Registry auth

`registry-url` writes a standard, neutral `.npmrc` — byte-for-byte the setup-node authutil contract — into `$RUNNER_TEMP/.npmrc` and exports `NPM_CONFIG_USERCONFIG`:

```ini
@scope:registry=https://npm.pkg.github.com/
//npm.pkg.github.com/:_authToken=${NODE_AUTH_TOKEN}
```

Set the token the same way you would for setup-node:

```yaml
- uses: nubjs/setup-nub@v1
  with:
    registry-url: https://npm.pkg.github.com
    scope: "@my-org"
- run: nub install
  env:
    NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Versioning

- `nubjs/setup-nub@v1` — floating major, gets fixes (recommended).
- `nubjs/setup-nub@v1.0.0` — pinned to a specific release.

## License

MIT
