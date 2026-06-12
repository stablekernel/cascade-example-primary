# cascade-example-primary

A worked example of a **cross-repo primary** managed by
[cascade](https://github.com/stablekernel/cascade). This repository runs its own
two-environment promotion pipeline (`staging` then `prod`) while also coordinating
artifacts produced in two separate source repositories.

## What This Repository Shows

This primary integrates with two source repositories using the two cross-repo
mechanisms cascade provides:

| Mechanism | How it works here |
|-----------|-------------------|
| **Synchronous reusable callback** | The `sharedlib` build calls a reusable workflow that lives in [`cascade-example-artifact-a`](https://github.com/stablekernel/cascade-example-artifact-a) (`uses: ...build-shared.yaml@main`). Its `artifact_id` output is captured during the orchestrate run. |
| **Asynchronous External Update** | After each source builds, it dispatches this repo's `external-update.yaml`. cascade writes `{deploy_name, sha, version, artifacts}` into the shared manifest under `state.<env>.external.<name>`. |

The `external` block in `.github/manifest.yaml` declares both sources:

- `cascade-example-artifact-a` -> deploy name `artifact-a`
- `cascade-example-artifact-b` -> deploy name `artifact-b`

A shared-manifest concurrency guard (`cancel_in_progress: false`) serializes
manifest writes so that two sources updating at the same time never lose an update.

## Layout

| Path | Purpose |
|------|---------|
| `.github/manifest.yaml` | Pipeline definition (envs, builds, deploys, `external` block, `release_token`). |
| `.github/workflows/orchestrate.yaml` | Generated. Builds and deploys to `staging` on every trunk commit. |
| `.github/workflows/promote.yaml` | Generated. Promotes `staging` to `prod` and publishes the release. |
| `.github/workflows/external-update.yaml` | Generated. Absorbs source updates into the shared manifest. |
| `.github/workflows/build-app.yaml`, `deploy-app.yaml` | Local callbacks for this repo's own application. |
| `.github/workflows/deploy-artifact-a.yaml`, `deploy-artifact-b.yaml` | Local callbacks tracking each external source. |
| `.github/workflows/scenario-suite.yaml` | End-to-end driver that validates the full cross-repo flow. |

## Getting Started

### Prerequisites

- A `CASCADE_STATE_TOKEN` repository secret with `contents: write` and
  `actions: write` scope (the latter lets the source repos dispatch this repo's
  External Update workflow).

### Triggering Pipelines

Push a change under `src/**` to start the orchestrate run, build the matrix, call
the external `sharedlib` callback, and deploy to `staging`. When `staging` is
verified, run **Actions -> Promote** to publish to `prod`.

## What the Test Driver Validates

`.github/workflows/scenario-suite.yaml` runs weekly (and on demand) and asserts:

| Job | Validates |
|-----|-----------|
| `reset` | Manifest state resets cleanly. |
| `merge-and-build` | Orchestrate runs, the synchronous external callback to artifact-a fires, `staging` is populated, and a prerelease is created. |
| `external-update-concurrent` | artifact-a and artifact-b dispatch External Update at the same time; both `{sha, version}` land under `state.staging.external.*` with no loss. |
| `promote-staging` | `prod` is populated, the release is published, and external source state survives the promote. |
