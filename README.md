# multi-branch-pages

Render a GitHub Pages site that contains docs for multiple branches at:

- `/` → generated index of rendered branches
- `/<branch>/...` → rendered docs for that branch

## MVP v1 decision: reusable workflow (`workflow_call`) over composite action

This repo implements MVP v1 as a **reusable workflow** at:

`/.github/workflows/publish-multi-branch-pages.yml`

### Why `workflow_call` is the better fit

- Publishing to GitHub Pages needs job-level permissions (`pages`, `id-token`) and deployment steps.
- The solution needs multiple jobs (discover branches, render matrix, assemble, deploy).
- The workflow can directly use `actions/upload-pages-artifact` and `actions/deploy-pages`.

### Trade-offs vs composite action

- Composite action pros: easy single-step reuse inside existing jobs.
- Composite action cons: cannot define jobs/permissions/environments itself, so callers must wire more complexity.
- Reusable workflow cons: callers invoke a workflow (not a single step), which is slightly less flexible.

For this product goal, reusable workflow is the cleaner MVP.

## Should this be a standalone repository?

**Yes — generally a good choice.**

Benefits:

- versioned distribution (`@v1`) across many repos
- independent release cadence
- centralized fixes and docs

Costs:

- cross-repo change management
- extra maintenance for compatibility and support

For a reusable Pages publisher, standalone distribution is usually the right default.

## Usage

In a consumer repository, enable GitHub Pages with **Build and deployment: GitHub Actions**, then call this workflow:

```yaml
name: Publish multi-branch docs

on:
  push:
    branches:
      - "**"
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  publish:
    uses: shielded-nate/multi-branch-pages/.github/workflows/publish-multi-branch-pages.yml@v1
    with:
      branch_whitelist: "*"
      render_command: ./scripts/render-docs.sh
      render_output_dir: docs/_site
```

## Interface contract for each branch

Each branch must support a common render command (`render_command`) that produces static files in `render_output_dir`.

## MVP behavior

- Branch whitelist supported via `branch_whitelist`:
  - `*` for all branches
  - or comma/newline/space-separated names
- Per-branch caching:
  - Cache key includes branch HEAD SHA
  - Unchanged branches restore from cache and skip rendering
- Removed branches:
  - Not included in newly assembled site output, so they disappear from published Pages content
- Index generation:
  - Creates root `index.html` with:
    - rendered branch links to `/<branch>/`
    - non-rendering branch links to each branch's GitHub code page
