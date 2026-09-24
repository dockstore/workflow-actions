# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Shared GitHub Actions for Dockstore repositories: reusable workflows (`on: workflow_call`) plus one composite action. There is no build, lint, or test suite here — the YAML is exercised only when a caller repository (e.g. a Dockstore Java/Maven project) invokes it. Callers reference these as `dockstore/workflow-actions/.github/workflows/<file>.yaml@<ref>` or `dockstore/workflow-actions/.github/actions/check-license@<ref>`.

## Components and how they fit together

- `.github/workflows/deploy_artifacts.yaml` — Maven deploy of a caller's project, optionally followed by a Docker image deploy.
  - `set_changelist` job computes the Maven `-Dchangelist` value from the git ref, and its output feeds both downstream jobs:
    - tag `1.16.0-alpha.0` → `.0-alpha.0` (tag must match a loose semver regex or the job fails)
    - `develop` branch → empty; the pom's own version is used unchanged
    - any other branch `feature/foo` → `.0-feature-foo-SNAPSHOT` (`/` replaced with `-`)
  - `deploy_maven` deploys with `./mvnw` to `central` on tags (bot `dockstore-bot`, secret `COLLAB_DEPLOY_TOKEN`) or `snapshots` otherwise (bot `dockstore-snapshot-bot`, secret `SNAPSHOT_DEPLOY_TOKEN`).
  - `deploy_image` runs only if `createDockerImage` is true and `quayRepository` is set; it calls `deploy_image.yaml` **pinned to `@main`** with `secrets: inherit`. So a change to `deploy_image.yaml` on a branch is not used by `deploy_artifacts.yaml` until it is merged to `main`.
- `.github/workflows/deploy_image.yaml` — builds the caller's project with `./mvnw clean install` (passing the changelist if given), then builds and pushes `quay.io/dockstore/<quayRepository>:<ref_name>` (with `/` replaced by `_`). It then uploads the image digest to S3 at `s3://$AWS_BUCKET/<ref>-<sha7>/<quayRepository>/image-digest.txt`, using AWS OIDC (`id-token: write`). It needs caller secrets `QUAY_USER`, `QUAY_TOKEN`, `AWS_ROLE_TO_ASSUME`, `AWS_REGION`, `AWS_BUCKET`. It accepts `buildArgs`, but `deploy_artifacts.yaml` does not currently expose or forward it.
- `.github/actions/check-license/action.yml` — composite action that diffs the working-tree copy of `licensefile` (default `THIRD-PARTY-LICENSES.txt`) against the committed version. The caller must regenerate the license file in an earlier step; a non-empty diff fails the job.

## Conventions

- Both workflows assume the caller has a Maven wrapper (`./mvnw`) at the repo root and use JDK `21.0.10+7.0.LTS` (temurin) on `ubuntu-22.04`. Keep these in sync across the two files when bumping them.
- Changing an input's name or meaning breaks every caller pinned to that ref, since callers usually pin `@main`.
