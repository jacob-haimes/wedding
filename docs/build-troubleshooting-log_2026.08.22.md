# Build Troubleshooting Log — GitHub Pages Deploy Failure

Detailed record of the investigation. See `BUILD_FIX_README.md` for the
short version.

## Symptom

Local `hugo` builds succeeded. Every GitHub Actions run of `Build` /
`Deploy website to GitHub Pages` failed. The person had previously worked
around similar issues by copying vendor partials into
`layouts/partials/blox/` and renaming them, but a prior attempt at that put
the repo into a state where *neither* local nor CI would build, and was
rolled back (`git log`: commit `08400ba "rollback cont"`).

## Environment mapping

| Location | Hugo version | Module cache |
|---|---|---|
| CI (`hugoblox.yaml` pin) | 0.136.5 | fresh per run, `actions/cache` keyed on `go.mod`/lockfiles |
| Local — apt package | 0.123.7+extended | shared `~/.cache/hugo_cache` |
| Local — `node_modules/hugo-extended` (used by `pnpm dev`/`pnpm build`) | 0.164.0 | shared `~/.cache/hugo_cache` |

Three different Hugos, none matching CI, sharing one cache. This is the
underlying reason "it builds locally" was not informative: depending on
which script/binary was invoked, the local build silently used a Hugo new
enough to tolerate module changes CI's pinned version couldn't handle.

## Repo structure relevant to the bug

- `hugoblox.yaml`: `build.hugo_version: '0.136.5'` — read by
  `.github/workflows/build.yml` via `yq` to select the Hugo version CI
  installs.
- `go.mod` / `go.sum`: Hugo Modules dependency graph (Go module system,
  reused by Hugo for its module mechanism).
- `config/_default/module.yaml`: declares which modules are *imported*
  (separate from, but must stay consistent with, `go.mod`).
- `.github/workflows/upgrade.yml`: scheduled (`cron: '0 5 * * 1'`, Mondays)
  job that runs `hugoblox@latest upgrade` and opens a PR bumping module
  versions in `go.mod`. **Does not** touch `hugoblox.yaml`'s pinned Hugo
  version. This is the mechanism that introduced the incompatible module
  versions in the first place.

## Diagnostic trail

### 1. Duplicate deploy workflows (investigated, not the cause)

`.github/workflows/publish.yaml` (legacy, hugo 0.136.5 hardcoded,
`actions/checkout@v4`, ends in `npx pagefind --source "public"`) and
`.github/workflows/deploy.yml` (current, calls `build.yml` as a reusable
workflow) both trigger on `push` to `main`, both named "Deploy website to
GitHub Pages," both use the `pages` concurrency group. Confirmed via the
Actions run history showing paired runs (e.g. `#7` and `#8`) on the same
commit SHA. `pagefind --source` is not valid syntax for pagefind ^1.4.0
(should be `--site`), so `publish.yaml` runs were failing independently of
the module issue below. Recommendation: delete `publish.yaml`. Not
pursued further in this session because the actual build-blocking error
was reproduced directly from `build.yml`'s logic.

### 2. First real error — `blox-analytics` partial not found

CI log:
```
ERROR render of "home" failed: ... executing "partials/site_head.html"
at <partialCached "blox-analytics/index" .>: error calling partialCached:
partial "blox-analytics/index" not found
```

`go.mod` had:
```
require github.com/HugoBlox/hugo-blox-builder/modules/blox-analytics v0.2.0 // indirect
```

Diffed the module's git tags directly
(`github.com/HugoBlox/hugo-blox-builder`):

- `blox-analytics/v0.1.3`: `layouts/partials/blox-analytics/index.html`
- `blox-analytics/v0.2.0`: `layouts/_partials/blox-analytics/index.html`

`layouts/_partials/` is a Hugo directory convention from the 0.146 era.
Hugo 0.136.5 (CI's pin) only scans `layouts/partials/`, so the v0.2.0
file is invisible to it regardless of whether the module is present.

`blox-tailwind@v0.3.1` (the module that calls the partial) itself
requires `blox-analytics v0.1.3`, confirmed via:
```
hugo mod graph | grep analytics
# github.com/HugoBlox/hugo-blox-builder/modules/blox-tailwind@v0.3.1
#   github.com/HugoBlox/hugo-blox-builder/modules/blox-analytics@v0.1.3
```

The `v0.2.0` pin in `go.mod` was an indirect requirement, most likely
written by a prior `upgrade.yml` run, that overrode MVS's natural choice
of v0.1.3.

**Fix attempted:** delete the `v0.2.0` require line, run `hugo mod tidy`.
Result: line was re-added, commented out
(`// require ... v0.2.0 // indirect`), and `go.sum` gained no `v0.1.3`
entries — `tidy` alone did not populate the checksums needed for a cold
build.

**Fix applied:** `hugo mod get
github.com/HugoBlox/hugo-blox-builder/modules/blox-analytics@v0.1.3`
directly, which wrote both `go.sum` lines needed:
```
github.com/HugoBlox/hugo-blox-builder/modules/blox-analytics v0.1.3 h1:...
github.com/HugoBlox/hugo-blox-builder/modules/blox-analytics v0.1.3/go.mod h1:...
```

### 3. Local toolchain didn't match CI, discovered mid-fix

Running a local build to verify the analytics fix (`hugo mod tidy` output)
surfaced:
```
WARN Module "github.com/HugoBlox/hugo-blox-builder/modules/blox-tailwind"
is not compatible with this Hugo version: Min 0.134.1 extended
```
`hugo version` at the time: `0.123.7+extended` (Ubuntu apt package),
below `blox-tailwind`'s declared minimum. This confirmed the local Hugo
in use did not match CI, and that "local build passes" could not be
trusted until that was corrected.

**Resolution:** installed Hugo 0.136.5 standalone, isolated from other
projects and from the shared module cache:

```bash
mkdir -p ~/.local/opt/hugo/0.136.5
# download+extract hugo_extended_0.136.5_linux-amd64.tar.gz there
```

Repo-local wrapper script `hugo136` (gitignored) sets `HUGO_CACHEDIR` to
a repo-local `.hugocache/` directory (also gitignored) before exec'ing
the pinned binary — avoids writing to `~/.cache/hugo_cache`, which is
shared across all Hugo projects on the machine, and avoids putting
anything on `PATH` that could shadow the system `hugo` for other work.

### 4. Second real error — `netlify` module, `hugo.Sites` not found

With `blox-analytics` resolved, next build error under 0.136.5:
```
Error: error building site: ... executing "index.redirects" at <hugo>:
can't evaluate field Sites in type interface {}
```
`hugo.Sites` diffed across the netlify module's tags
(`github.com/HugoBlox/kit`):
- `modules/integrations/netlify/v1.2.1`: `range $page := where site.AllPages ...`
- `modules/integrations/netlify/v1.3.0`: `range $s := hugo.Sites` (function not in 0.136.5)

This module ships no `hugo.yaml`/version constraint, so it produced no
compatibility warning, only a runtime failure. It generates
Netlify-specific `_headers`/`_redirects` output formats, irrelevant to a
GitHub Pages deployment.

**Fix attempted:** downgrade the require line to v1.2.1 via `hugo mod
get ...@v1.2.1`. Result: next invocation of `hugo` (not even a rebuild,
just module collection) logged `go: upgraded ... v1.2.1 => v1.3.0` and
reverted the pin. Root cause of *that* traced to `config/_default/module.yaml`
still declaring the netlify import independently of `go.mod` — Hugo's
module loader reconciles configured imports against `go.mod` and appears
to re-resolve/upgrade in that process regardless of an explicit `require`
pin.

**Fix applied:** removed the module entirely —
- deleted the import line from `config/_default/module.yaml`
- removed `headers`/`redirects` from `outputs.home` in
  `config/_default/hugo.yaml` (module owned those output-format
  definitions; leaving them declared with the module gone would error on
  undefined formats)
- deleted `netlify.toml` (Netlify-specific config, unused for Pages
  deployment)
- removed the `require` line for the netlify module from `go.mod`

### 5. `hugo mod tidy` / `hugo mod graph` unexpectedly upgrading `blox-tailwind`

After the netlify removal, running `hugo mod tidy` (to clean up `go.mod`)
had a second, more serious side effect: it re-resolved the *entire*
dependency graph from the still-live `blox-tailwind` import and picked
`v0.10.0` (latest), rewriting `go.mod` from `v0.3.1` to `v0.10.0` without
being asked to. That version then failed differently:
```
Error: failed to load modules: invalid module config for
"...blox-tailwind": "open .../blox-tailwind@v0.10.0/hugo_stats.json:
permission denied"
```
(Cause of that specific permission error not conclusively identified —
plausibly the module's own `build.writeStats` config attempting to write
inside the read-only module cache tree. Not pursued, since v0.10.0 was
not the intended version regardless.)

Subsequent attempts to re-pin `blox-tailwind` to `v0.3.1` via a plain
`require` line were **also** silently overwritten on the very next
invocation of `hugo mod graph` (a nominally read-only inspection command),
confirming this isn't limited to `tidy` — something in Hugo's module
loading step performs an unconstrained upgrade of this specific module on
every run, independent of the command issued.

**Fix applied:** a `replace` directive, which Go's tooling cannot
renegotiate the way it can a bare `require`:
```
require github.com/HugoBlox/hugo-blox-builder/modules/blox-tailwind v0.3.1
replace github.com/HugoBlox/hugo-blox-builder/modules/blox-tailwind => github.com/HugoBlox/hugo-blox-builder/modules/blox-tailwind v0.3.1
```
Confirmed working: `go.mod`'s `require` line still gets rewritten to
`v0.10.0` on each invocation (expected Go behavior — with a `replace`
present, the `require` line records the pre-replace MVS resolution as
bookkeeping, while the `replace` determines what's actually used).
`hugo mod graph` output confirmed the substitution is applied:
```
project github.com/.../blox-tailwind@v0.10.0 => github.com/.../blox-tailwind@v0.3.1
```

A leftover artifact from the earlier bad `v0.10.0` fetch (read-only files
under `.hugocache/`, a deliberate Go module-cache protection, not
corruption) briefly blocked `rm -rf .hugocache`; resolved with `chmod -R
u+w .hugocache && rm -rf .hugocache` before the next clean build.

### 6. Confirmed fix, first on warm local cache, then cold

```bash
rm -rf public
./hugo136 --minify   # exit 0, warm .hugocache
```
Then, the test that actually predicts CI — fresh clone, throwaway cache:
```bash
cd /tmp && rm -rf citest
git clone --branch fix/blox-analytics-pin ~/Git-projects/wedding citest
cd citest
HUGO_CACHEDIR=$(mktemp -d) HUGO_ENVIRONMENT=production ~/.local/opt/hugo/0.136.5/hugo --minify
# exit 0, no netlify module downloaded, 17 pages rendered
```
This is the point at which the fix was considered verified — a clone
containing only committed state, no shared cache, no residual local
config.

### 7. Ancillary findings, addressed alongside

- **`languages.en.locale`**: deprecated since Hugo 0.112.0, warning-only
  through 0.136.5, becomes a fatal `ERROR` at 0.137.0. Fix: move under
  `languages.en.params.locale` in `config/_default/languages.yaml`.
- **`baseURL: 'leahandjacobwedding.com'`** (in `config/_default/hugo.yaml`)
  has no scheme. Not currently causing failures because
  `build.yml` overrides it at build time
  (`hugo --baseURL "${{ steps.pages.outputs.base_url }}/"`), but incorrect
  as a standalone value.
- **`public/` diff churn**: an interrupted local build attempt (Hugo
  0.123.7, before the version-pinning fix, killed partway through)
  left `public/` — which is tracked in git — with ~100 stale
  deletions/renames relative to the committed copy, plus untracked files
  using a different image-hash naming scheme than 0.136.5 produces.
  Cosmetic (deploy always rebuilds `public/` fresh in CI, the tracked
  copy is never read), but recommended either discarding
  (`git checkout -- public && git clean -fd public`) or untracking it
  entirely as a build artifact, alongside `hugo_stats.json` and
  `.hugo_build.lock`:
  ```bash
  git rm -r --cached public hugo_stats.json .hugo_build.lock
  printf 'public/\nhugo_stats.json\n.hugo_build.lock\n' >> .gitignore
  ```
  `node_modules/` was untracked the same way during this investigation
  (1,096 tracked files at the time, all reinstalled by `pnpm install` in
  CI regardless of what's committed).

- **`upgrade.yml`**: identified as the origin of both breaking module
  bumps (`blox-analytics` v0.2.0, `netlify` v1.3.0), since it updates
  `go.mod` module pins on a schedule without correspondingly updating
  `hugoblox.yaml`'s Hugo version pin. Recommended for deletion to prevent
  recurrence; not deleted as part of this investigation, left as a
  follow-up action.

## Deferred: full Hugo/Tailwind upgrade

An alternative fix — raising `hugoblox.yaml`'s Hugo pin to 0.164.0 (matching
`node_modules/hugo-extended`) instead of downgrading modules — was
considered and deliberately not pursued in this session. Reasons:
- `blox-tailwind@v0.3.1` is a Tailwind v3 / PostCSS-era module (its own
  `package.json` requires `tailwindcss ^3.3.5`, `postcss-cli`,
  `autoprefixer`), while this repo's `package.json` is already the
  Tailwind v4 starter (`@tailwindcss/cli ^4.1.12`, no PostCSS deps). A
  Hugo-only bump doesn't resolve this mismatch; it would need
  `blox-tailwind` bumped too (untested at that version in this repo).
- Four custom partials in `layouts/partials/blox/`
  (`hero-with-stats-2.html`, `travel.html`, `sectioned-list.html`,
  `markdown-2.html`) were written against `blox-tailwind@v0.3.1`'s
  partial signatures. A module version bump that changes those
  signatures could break the overrides silently (wrong rendering, not a
  build error), which is harder to catch than the failures diagnosed
  above.

If pursued later: same pinned-binary-and-isolated-cache technique
applies, install to `~/.local/opt/hugo/0.164.0/` with a second wrapper
(e.g. `hugo164`), and validate on a separate branch via the PR-triggered
`build.yml` before merging.
