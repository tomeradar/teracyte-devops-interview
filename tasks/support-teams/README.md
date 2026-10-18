# Support Dev and Data/ML delivery

**Required task** · [Assignment](../../README.md)

## Goal

Support application engineers and data scientists with independent, reliable delivery paths.

## Requirements

- Add .github/workflows/ci-core-dev.yml for API/Celery changes and .github/workflows/ci-data-ml.yml for analysis-step and model changes.
- Run relevant builds/tests in each workflow. The Data/ML workflow must run the E2E verification; shared runtime/configuration changes must trigger both teams' relevant checks.
- Use path filters, job conditions or workflow_dispatch inputs so an API-only change does not release a model and an analysis-only change does not rebuild the API unnecessarily.
- Make model selection configurable through a path/version and expected digest. A model-only run must validate the selected artifact without changing application code, rebuilding the API, or baking weights into an image.
- Reuse local commands, propagate failures and retain useful logs on failure.
- Briefly explain how each team would promote and roll back a change. Actual publishing, deployment and cloud credentials are optional.

## Suggested starting point

Core Dev owns services/api; Data/ML owns pipelines/steps and model configuration. Classify shared files such as Compose, shared Dockerfiles, dependencies and scripts explicitly. With one shared image, shared-image changes may legitimately rebuild both consumers.

The starter manifest already records a model path, and inference records SHA-256. A configured local model file is enough; an ML registry and a new model algorithm are not required. A second dummy artifact can demonstrate model selection.

Keep the two workflows small and reuse the [verification command](../verify-experiment/README.md). A future UI path may be documented; do not build a frontend for this exercise.

## How to demonstrate it

Record workflow runs and a short change-routing table covering:
- API/Celery-only changes.
- Analysis-step-only changes.
- A model-only selection with its expected digest.
- A shared configuration change.

Show which checks/builds run and why. If Actions cannot run because of repository permissions, supply workflow files, local command output and an explicit limitation.

## Optional improvements

Publish immutable images, add approval-based promotion, cache builds, or add separate deployment environments. Avoid adding infrastructure solely to demonstrate a release.
