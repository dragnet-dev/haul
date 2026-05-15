# Dragnet Engine — Pre-Launch Cleanup Checklist

These are the changes required in `dragnet-dev/dragnet` (the engine repo) before going live
with the split. The data/rules side now lives here in `dragnet-dev/haul`.

Work through these in order — each step depends on the previous one.

---

## 1. Remove the committed binary

The compiled `dragnet` binary (18 MB) is currently tracked by git. It should never be committed.

```bash
git rm --cached dragnet
```

---

## 2. Remove committed data directories

The following directories are generated output and belong in `haul`, not in the engine repo.
Remove them from git tracking without deleting the local files (you may still want them for
local dev runs):

```bash
git rm -r --cached supply/
git rm -r --cached malware/
git rm -r --cached ransomware/
git rm -r --cached cve/
git rm -r --cached feeds/
git rm -r --cached incidents/
git rm -r --cached state/
```

> **Note on `state/`:** The OSSF cache at `state/ossf-cache/` is already gitignored, but the
> `state/` directory itself may have other tracked contents (e.g. `last_sync.json` files).
> Run `git status state/` first to see what is currently tracked before removing.

---

## 3. Remove `actors/` from the engine repo

Actor profiles are data, not engine code. They now live in `haul/actors/` and are read by the
dragnet binary at runtime from its working directory (which in CI is the checked-out haul repo).

```bash
git rm -r --cached actors/
```

---

## 4. Update `.gitignore`

The current `.gitignore` only excludes the binary and the OSSF cache. Add everything else
that should never be committed to the engine repo:

```gitignore
# Compiled binary
dragnet
dragnet-linux-amd64
dragnet-darwin-amd64
dragnet-darwin-arm64
dragnet-windows-amd64.exe

# Generated data — lives in dragnet-dev/haul
supply/
malware/
ransomware/
cve/
feeds/
incidents/
state/
actors/

# OSSF malicious-packages git cache (~1.8 GB shallow clone)
state/ossf-cache/
```

---

## 5. Rename `dragnet.yaml` → `dragnet.yaml.example`

The live config lives in `haul/dragnet.yaml`. The engine repo should only carry a documented
template so new users know what fields are available:

```bash
git mv dragnet.yaml dragnet.yaml.example
git commit -m "chore: dragnet.yaml → dragnet.yaml.example (live config lives in haul)"
```

Strip or redact any API keys in the example before committing.

---

## 6. Commit the removals

```bash
git commit -m "chore: remove generated data and actors — moved to dragnet-dev/haul"
git push
```

---

## 7. Replace GitHub Actions workflows

The engine repo currently has three workflows that do too much. They need to be split:

### 7a. Add `build.yml` (new)

Dragnet needs a release workflow that compiles the binary and publishes it to GitHub Releases
whenever a `v*` tag is pushed. The `haul` sync workflow downloads from here.

Create `.github/workflows/build.yml`:

```yaml
name: Build and Release

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:

permissions:
  contents: write

jobs:
  build:
    name: Build ${{ matrix.asset }}
    runs-on: ubuntu-latest
    strategy:
      matrix:
        include:
          - goos: linux
            goarch: amd64
            asset: dragnet-linux-amd64
          - goos: darwin
            goarch: amd64
            asset: dragnet-darwin-amd64
          - goos: darwin
            goarch: arm64
            asset: dragnet-darwin-arm64
          - goos: windows
            goarch: amd64
            asset: dragnet-windows-amd64.exe

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.26'
          cache: true

      - name: Build
        env:
          GOOS: ${{ matrix.goos }}
          GOARCH: ${{ matrix.goarch }}
        run: |
          go build -o "${{ matrix.asset }}" \
            -ldflags="-X main.version=${{ github.ref_name }}" \
            .

      - name: Upload to release
        uses: softprops/action-gh-release@v2
        with:
          files: ${{ matrix.asset }}
          generate_release_notes: true
```

### 7b. Update `sync.yml` — remove it (move to haul)

The current `sync.yml` builds the binary inline and then runs sync/generate/commit. That
entire workflow now lives in `haul/.github/workflows/sync.yml`. Delete it from the engine:

```bash
git rm .github/workflows/sync.yml
```

### 7c. Update `validate-pr.yml` — swap build for binary download

The current validate-pr builds from source. Update it to download the latest release instead:

```yaml
- name: Download Dragnet binary
  run: |
    gh release download \
      --repo dragnet-dev/dragnet \
      --pattern dragnet-linux-amd64 \
      --output dragnet
    chmod +x dragnet
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Remove the `actions/setup-go` step and the `go build` step.

### 7d. Remove `auto-merge.yml` (move to haul)

Auto-merge logic operates on incident PRs which all live in haul now. Delete it from the engine:

```bash
git rm .github/workflows/auto-merge.yml
```

---

## 8. Publish the first release

Before haul's sync workflow can run, there must be at least one published release with binaries attached.

```bash
git tag v0.1.0 -m "Initial release"
git push origin v0.1.0
```

Wait for the `build.yml` workflow to complete and confirm binaries appear on the GitHub Releases page.

Then trigger haul's sync manually to verify end-to-end:

```
dragnet-dev/haul → Actions → Dragnet Sync → Run workflow
```

---

## Summary of what stays in `dragnet-dev/dragnet`

```
cmd/
internal/
main.go
go.mod
go.sum
dragnet.yaml.example
schema/
.github/workflows/
  build.yml        ← new
  validate-pr.yml  ← updated (download binary, no go build)
README.md
```

Everything else — data, config, actors, rules, feeds, incidents — lives in `dragnet-dev/haul`.
