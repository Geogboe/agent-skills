---
name: github-release-expert
description: Set up or migrate a GitHub release pipeline that uses Release Please for automated release PRs, versioning, tags, and GitHub Releases, and optional artifact publishing. Use when asked to automate releases on push/merge, replace manual tag-based release flows, fix Release Please permission failures, or validate end-to-end release behavior in GitHub.
---

# Setup Release Please Pipeline

Build a reliable release pipeline with three layers:
1. CI workflow for tests/build checks
2. Release Please workflow to create/update release PRs and cut releases
3. Optional publisher workflow to attach artifacts to releases

## Workflow

1. Audit current release setup.
   - Identify existing workflows, trigger conditions, and manual release steps.
   - Capture current artifact naming so migration does not break install docs.

2. Create or update CI workflow.
   - Trigger on `push` and `pull_request` to the default branch.
   - Run at least: dependency download, lint/vet equivalent, tests, and build checks.
   - Keep CI independent from release publishing.

3. Add Release Please configuration.
   - Add `release-please-config.json` and `.release-please-manifest.json`.
   - Configure the package path(s), release type, and initial version.
   - Enable prerelease mode when requested.

4. Add Release Please workflow.
   - Trigger on pushes to the default branch and optional `workflow_dispatch`.
   - Set permissions:
     - `contents: write`
     - `pull-requests: write`
   - Run `googleapis/release-please-action@v4` with config + manifest files.

5. Add optional artifact publisher workflow/job.
   - Run only when a release is created.
   - Check out the release tag and publish artifacts.
   - Keep publishing logic separated from version calculation logic.

6. Update release docs.
   - Replace "create/push tag manually" instructions with "merge release PR" flow.
   - Document required commit style (Conventional Commits).
   - Add troubleshooting for permissions and workflow failures.

7. Validate end-to-end.
   - Validate local lint/test/build.
   - Push changes and verify workflows in GitHub Actions.
   - Confirm release PR creation, merge, tag creation, release creation, and artifact upload.

## Required GitHub Settings

Set repository Actions permissions so Release Please can open release PRs:
- Repository Settings -> Actions -> General -> Workflow permissions: **Read and write permissions**
- Enable: **Allow GitHub Actions to create and approve pull requests**

If this is not enabled, Release Please can fail with: "GitHub Actions is not permitted to create or approve pull requests."

## Release Numbering Guidance

- For normal semver start: set `initial-version` in Release Please config (for example `0.1.0`).
- For prerelease tracks, enable prerelease mode in config and decide whether the first published tag should be stable (`v0.1.0`) or prerelease (`v0.1.0-<id>.1`).
- If the wrong release type is created, update the GitHub release metadata and adjust config before the next cut.

## Go Artifact Publishing (Optional)

For Go projects, read `references/go-goreleaser.md`.
