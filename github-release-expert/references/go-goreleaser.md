# Go + GoReleaser Integration

Use this reference when the repository is a Go project and release artifacts should be attached automatically.

## Recommended Files

- `.goreleaser.yaml`
- `.github/workflows/release-please.yml` (Release Please + GoReleaser jobs)
- `.github/workflows/test.yml` (CI)

## Example `.goreleaser.yaml`

Use a baseline config similar to:

```yaml
version: 2

project_name: myapp

before:
  hooks:
    - go mod tidy

builds:
  - id: myapp
    main: .
    binary: myapp
    env:
      - CGO_ENABLED=0
    goos:
      - linux
      - darwin
      - windows
    goarch:
      - amd64
      - arm64
    ignore:
      - goos: windows
        goarch: arm64

archives:
  - id: myapp-archives
    builds:
      - myapp
    name_template: "{{ .ProjectName }}-{{ .Version }}-{{ .Os }}-{{ .Arch }}"
    format_overrides:
      - goos: windows
        format: zip

checksum:
  name_template: checksums.txt

changelog:
  use: github-native

release:
  prerelease: auto
```

Adjust `project_name`, `binary`, and target matrix for the repository.

## Example Release Workflow Pattern

Use two jobs in one workflow:

1. `release-please` job
   - Runs Release Please action on push to default branch
   - Emits outputs like `release_created` and `tag_name`

2. `goreleaser` job
   - `needs: release-please`
   - `if: needs.release-please.outputs.release_created == 'true'`
   - Checks out `${{ needs.release-please.outputs.tag_name }}` with `fetch-depth: 0`
   - Runs `goreleaser/goreleaser-action@v6` with `args: release --clean`

## CI Recommendations for Go Repos

- Use `actions/setup-go` with `go-version-file: go.mod`.
- Keep CI fast and independent from release publishing.
- Typical CI checks: `go vet ./...`, `go test ./...`, cross-platform build smoke tests.

## Common Failure Modes

1. Release Please cannot create PRs
   - Fix repository Actions permissions (read/write + allow PR creation/approval).

2. GoReleaser not running
   - Check `if` condition and ensure release was actually created by Release Please.

3. Artifact names break install docs
   - Align GoReleaser `archives.name_template` with documented download URLs.

4. Local tests fail in restricted environments
   - Set writable cache when needed (for example `GOCACHE=/tmp/go-build-cache`).
