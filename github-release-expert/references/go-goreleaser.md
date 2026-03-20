# Go + GoReleaser Integration

Use this reference when the repository is Go and release assets should be attached automatically.

## Recommended File Layout

```
.github/workflows/
  ci.yml              — lint, test, build, security scans (independent of release)
  release-please.yml  — release-please job + goreleaser job (same file, two-job pattern)
.goreleaser.yml
release-please-config.json
.release-please-manifest.json
```

## CI Workflow — Known Gotchas

### golangci-lint: use v2 module path, build from source

Do NOT use `golangci-lint-action` with `version: latest` if the project targets a newer Go version. The pre-built golangci-lint binary lags behind Go releases; if `go.mod` targets Go 1.24+ you'll get:

```
can't load config: the Go language version (go1.24) used to build golangci-lint
is lower than the targeted Go version (1.25.x)
```

Fix — build from source with the same Go version the project uses:

```yaml
- uses: actions/setup-go@v5
  with:
    go-version-file: go.mod
- name: Install golangci-lint
  run: go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest
- name: Run golangci-lint
  run: golangci-lint run ./...
```

Note the module path: `golangci-lint/v2/cmd/golangci-lint` — using the v1 path silently installs v1 which rejects v2 config files with `you are using a configuration file for golangci-lint v2 with golangci-lint v1`.

### golangci-lint v2 config structure

The `.golangci.yml` config file requires `version: "2"` at the top and uses different syntax:

```yaml
version: "2"

linters:
  enable:
    - errcheck
    - govet
    - staticcheck
    - gosec
    - unused
    - gocritic
    - misspell

  settings:              # was "linters-settings:" in v1
    gosec:
      excludes:
        - G204           # subprocess via variable (intentional in CLI tools)
        - G304           # file inclusion via variable (intentional in loaders)

  exclusions:            # was "issues.exclude-rules:" in v1
    rules:
      - path: _test.go
        linters:
          - errcheck
          - gosec

formatters:              # gofmt/goimports are formatters, NOT linters in v2
  enable:
    - gofmt
```

### gitleaks: use CLI, not the GitHub Action

`gitleaks/gitleaks-action@v2` requires a paid `GITLEAKS_LICENSE` secret for private repositories. Instead, install the CLI binary directly (works on all repo visibility levels):

```yaml
- name: Install gitleaks
  run: |
    gh release download --repo gitleaks/gitleaks \
      --pattern "*linux_x64.tar.gz" -O - | tar -xz -C /usr/local/bin gitleaks
  env:
    GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
- name: Run gitleaks
  run: gitleaks detect --source=. --verbose --redact
```

### govulncheck (recommended addition)

Add the official Go vulnerability scanner — it checks against the Go vuln database and is fast:

```yaml
- name: Install govulncheck
  run: go install golang.org/x/vuln/cmd/govulncheck@latest
- name: Run govulncheck
  run: govulncheck ./...
```

## Release Please Workflow — Two-Job Pattern

**Always use the two-job pattern** (release-please + goreleaser in the same file). See `SKILL.md` for explanation of why separate files silently break goreleaser.

```yaml
name: Release Please

on:
  push:
    branches: [main]

permissions:
  contents: write
  pull-requests: write

jobs:
  release-please:
    runs-on: ubuntu-latest
    outputs:
      release_created: ${{ steps.release.outputs.release_created }}
      tag_name: ${{ steps.release.outputs.tag_name }}
    steps:
      - uses: googleapis/release-please-action@v4
        id: release
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json

  goreleaser:
    needs: release-please
    if: needs.release-please.outputs.release_created == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ needs.release-please.outputs.tag_name }}  # must be the tag, not HEAD
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
      - name: Install syft (required for SBOM generation)
        uses: anchore/sbom-action/download-syft@v0        # goreleaser-action does NOT auto-install syft
      - uses: goreleaser/goreleaser-action@v6
        with:
          args: release --clean
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**Critical details:**
- `ref: ${{ needs.release-please.outputs.tag_name }}` — goreleaser validates that the tag points to HEAD; without this it fails with `"git tag vX.Y.Z was not made against commit <sha>"`
- `anchore/sbom-action/download-syft@v0` — goreleaser-action v6 does NOT auto-install syft even when `.goreleaser.yml` requests SBOMs; the step must be explicit

## Baseline `.goreleaser.yml`

```yaml
version: 2

before:
  hooks:
    - go mod tidy
    - go generate ./...

builds:
  - id: myapp
    main: ./cmd/myapp
    binary: myapp
    env:
      - CGO_ENABLED=0
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]
    ignore:
      - goos: windows
        goarch: arm64
    ldflags:
      - -s -w -X main.version={{.Version}}

archives:
  - formats: [tar.gz]
    format_overrides:
      - goos: windows
        formats: [zip]
    name_template: "{{ .ProjectName }}_{{ .Version }}_{{ .Os }}_{{ .Arch }}"
    files:
      - LICENSE
      - README.md

checksum:
  name_template: "checksums.txt"

sboms:
  - artifacts: archive   # requires syft on PATH — see workflow above

changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
      - "^chore:"
      - Merge pull request
      - Merge branch

release:
  draft: false
  prerelease: auto       # marks 0.x.x as pre-release automatically
```

## Release Please Config for Go Repos

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "release-type": "go",
  "packages": {
    ".": {
      "release-type": "go",
      "include-component-in-tag": false,
      "include-v-in-tag": true,
      "changelog-sections": [
        {"type": "feat",     "section": "Features"},
        {"type": "fix",      "section": "Bug Fixes"},
        {"type": "refactor", "section": "Refactoring"},
        {"type": "perf",     "section": "Performance"},
        {"type": "docs",     "section": "Documentation", "hidden": true},
        {"type": "chore",    "section": "Miscellaneous",  "hidden": true}
      ]
    }
  }
}
```

`.release-please-manifest.json` — set to current version:
```json
{ ".": "0.1.0" }
```

## Local CI Parity (Taskfile)

Mirror CI checks locally to catch failures before pushing:

```yaml
# Taskfile.yml
lint:
  cmds:
    - golangci-lint run ./...

vuln:
  cmds:
    - govulncheck ./...

secrets:
  cmds:
    - gitleaks detect --source=. --verbose --redact

ci:
  cmds:
    - task: secrets
    - task: lint
    - task: vuln
    - task: test
    - task: build
```

## Common Failure Modes

1. **Release created but no assets attached.**
   - Using separate goreleaser workflow — GITHUB_TOKEN can't trigger other workflows.
   - Fix: two-job pattern in same file (see above).

2. **Release Please cannot create PRs.**
   - Repo Actions permissions are read-only.
   - Fix: `gh api repos/OWNER/REPO/actions/permissions/workflow --method PUT -F default_workflow_permissions=write -F can_approve_pull_request_reviews=true`

3. **GoReleaser fails: "tag not made against this commit".**
   - Goreleaser is running against HEAD but the tag points to an earlier commit.
   - Fix: `ref: ${{ needs.release-please.outputs.tag_name }}` in checkout.

4. **GoReleaser fails: "exec: syft: not found".**
   - SBOM generation requires syft; goreleaser-action doesn't install it automatically.
   - Fix: add `anchore/sbom-action/download-syft@v0` step before goreleaser.

5. **golangci-lint fails with Go version mismatch.**
   - Pre-built golangci-lint binary was built with an older Go than the project targets.
   - Fix: `go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@latest`

6. **golangci-lint fails: "config file for v2 with golangci-lint v1".**
   - `go install` resolved the v1 module path.
   - Fix: use `golangci-lint/v2/cmd/golangci-lint` (the v2 module path).

7. **gitleaks fails with license error on private repo.**
   - `gitleaks/gitleaks-action@v2` requires paid license for private repos.
   - Fix: install gitleaks CLI via `gh release download` (see CI section above).

8. **govulncheck fails on standard library CVEs.**
   - Pin workflows to a patched Go patch version.
