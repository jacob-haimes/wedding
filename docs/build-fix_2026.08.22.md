# GitHub Pages Build Failure — Summary

## What was happening

The site built locally but failed on every GitHub Actions build/deploy.

## Root cause

`hugoblox.yaml` pins CI to **Hugo 0.136.5**. The `Upgrade HugoBlox` scheduled
workflow (`.github/workflows/upgrade.yml`, runs weekly) had bumped module
versions past what 0.136.5 supports, without also bumping the Hugo pin:

1. **`blox-analytics` v0.2.0** moved its templates from `layouts/partials/`
   to `layouts/_partials/` — a directory convention Hugo only recognizes
   from 0.146+. Result: `partial "blox-analytics/index" not found`.
2. **`kit/modules/integrations/netlify` v1.3.0** uses the `hugo.Sites`
   template function, also unavailable before 0.146. Result:
   `can't evaluate field Sites in type interface {}`. This module produces
   Netlify-only `_headers`/`_redirects` files — not needed for GitHub Pages.

Locally, the failure wasn't visible because the machine had **three
different Hugo binaries** in play (apt 0.123.7, npm `hugo-extended` 0.164.0,
none matching CI's 0.136.5), so a "local pass" carried no information about
whether CI would pass.

## The fix

- Pinned `blox-tailwind` (which pulls in `blox-analytics`) to v0.3.1 via a
  `replace` directive in `go.mod` — a plain `require` pin was not enough;
  something in Hugo's module loading kept re-upgrading it.
- Removed the `netlify` integration module entirely (import in
  `config/_default/module.yaml`, output formats in `hugo.yaml`,
  `netlify.toml`).
- Deleted `.github/workflows/upgrade.yml` so this can't silently recur.
- Moved `languages.en.locale` under `languages.en.params` (deprecated,
  becomes a fatal error at Hugo 0.137.0).

## Key commands

Install a Hugo binary matching CI, isolated from other projects (no PATH
change, no shared `~/.cache/hugo_cache`):

```bash
mkdir -p ~/.local/opt/hugo/0.136.5 && cd /tmp
wget https://github.com/gohugoio/hugo/releases/download/v0.136.5/hugo_extended_0.136.5_linux-amd64.tar.gz
tar xzf hugo_extended_0.136.5_linux-amd64.tar.gz hugo
mv hugo ~/.local/opt/hugo/0.136.5/hugo && chmod +x ~/.local/opt/hugo/0.136.5/hugo
```

In the repo, `./hugo136` wraps that binary with its own cache dir
(`.hugocache/`, gitignored) so it never touches the shared cache:

```bash
cat > hugo136 <<'EOF'
#!/usr/bin/env bash
export HUGO_CACHEDIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)/.hugocache"
exec "$HOME/.local/opt/hugo/0.136.5/hugo" "$@"
EOF
chmod +x hugo136
```

Build with it exactly as CI would:

```bash
rm -rf public
./hugo136 --minify ; echo "exit: $?"
```

The test that actually matters — a cold clone with a throwaway cache, no
warm state from your machine:

```bash
cd /tmp && rm -rf citest
git clone --branch <branch> ~/Git-projects/wedding citest
cd citest
HUGO_CACHEDIR=$(mktemp -d) HUGO_ENVIRONMENT=production ~/.local/opt/hugo/0.136.5/hugo --minify
```

## Safety net used

```bash
git tag known-good-YYYYMMDD   # immutable restore point, made before any changes
git push origin --tags
```

Rollback if needed: `git reset --hard known-good-YYYYMMDD`.

## Still open / worth doing later

- Untrack remaining build artifacts — none of these are read by CI,
  they're only bloating the repo and producing noisy diffs after local
  builds (`node_modules/` already untracked):
  ```bash
  git rm -r --cached public hugo_stats.json .hugo_build.lock
  printf 'public/\nhugo_stats.json\n.hugo_build.lock\n' >> .gitignore
  ```
- `baseURL: 'leahandjacobwedding.com'` in `config/_default/hugo.yaml` has
  no scheme (`https://`) — CI overrides it at build time, so not currently
  breaking anything, but worth fixing for correctness.
- Duplicate deploy workflows: `.github/workflows/publish.yaml` (legacy) and
  `deploy.yml` both trigger on push to `main`. `publish.yaml` uses a
  `pagefind --source` flag that doesn't exist in current pagefind — delete
  it.
- Full upgrade to Hugo 0.164.0 + `blox-tailwind` v0.10.0+ was deferred as a
  separate, non-urgent piece of work (Tailwind v3→v4 migration for that
  module, plus re-validating four custom Blox overrides in
  `layouts/partials/blox/`).
