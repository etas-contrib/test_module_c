# Design: Module → bazel_registry Publication

This document describes how modules publish new versions to the shared
`bazel_registry` via GitHub Actions. It uses **three workflows**:

1. A **reusable workflow** in the registry/cicd-workflows repo
2. An **automatic workflow** in each module repo. Especially for repos that do not perform any post PR testing.
3. A **manual / recovery workflow** in each module repo

---

## Goals

- **One way** for modules to request a registry update.
- As much logic as possible in the **reusable** workflow.
- **Secure** handling of tokens

---

## 1. Reusable workflow (in `bazel_registry` / `cicd-workflows` repo)

**File:**  
`reusable-bazel-registry-publish.yml`

**Called from:** module repos (`workflow_call`).

### Responsibilities

- Determine and validate the **tag**:
  - `push` event: tag from `github.ref` (`refs/tags/v*`).
  - `release` event: tag from `github.event.release.tag_name`.
  - `workflow_dispatch`: tag from input `version` → `v<version>`.
- Validate version:
  - Strip leading `v`, enforce **PEP 440** via `packaging.version.Version` (TODO: which `semver` variant do we use?).
  - On `release` events: ensure release name starts with the tag.
- Determine **Bazel module name**:
  - Checkout caller repo.
  - Parse `MODULE.bazel` for `module(name = "…")`.
- Create / update **PR in `bazel_registry`**:
  - Checkout `BAZEL_REGISTRY_REPOSITORY` (org var, default: `eclipse-score/bazel_registry`).
  - Install and run `registry_manager`, e.g.:

    ```bash
    python registry_manager add "<module>" "<tag>" "<source_repo>" "<source_sha>"
    ```

  - Use `peter-evans/create-pull-request@v7` to:
    - create/update branch `update/<module>/<tag>`
    - commit changes
    - open/update PR with title `Add <module> @ <tag>`.

### Inputs & Secrets

- **Inputs**
  - `version` (optional): PEP 440 version without `v`, only used for manual runs.
- **Secrets**
  - `registry-token` (required): PAT with write access to `bazel_registry`
    (used for checkout + PR creation).

---

## 2. Auto publish workflow (in module repo)

**File:**  
`.bazel-registry-auto.yml`

**Purpose:**  
Publish module to registry automatically on **tag pushes** and **releases**.

### Triggers

- `push` with `tags: v*`
- `release` with `types: [published]`

## Alternatives Considered / Open Points

- Should modules create a PR or trigger a workflow? Triggering the workflow is a clearer API, but creating a PR creates a true end-to-end experience. The created PR can be linked.
- Should we even support on tag / on release?
