# approval-demo

Demo repository for GitHub Actions CD orchestration patterns.

## Orchestrator Design

### Channel Flow

```
server-demo/build.yml (push: rc)
  └─→ creates release/<version> branch in approval-demo
  └─→ triggers orchestrator-rc.yml on release/<version>

orchestrator-rc.yml
  pull-artifacts
  └─→ approve-releasable-pipeline  [gate: Approve-Pipeline-RC]
        ├─→ deploy-eu-qa-db → deploy-eu-qa-app
        └─→ deploy-us-qa-db → deploy-us-qa-app
              └─→ pre-release-prd  [gate: PreRelease-PRD]
                    └─→ release → publish
                          └─→ deploy-eu-prd-db → deploy-eu-prd-app
                                └─→ deploy-us-prd-db → deploy-us-prd-app
```

Hotfix channel (`orchestrator-hotfix-rc.yml`) skips QA entirely — `pre-release-prd` needs only `approve-releasable-pipeline`.

## Key Design Decisions

### workflow_call over workflow_dispatch for deploy steps
`workflow_call` gives an integrated job graph (all steps visible in one workflow run), native environment gate UX, and proper job-level status. `workflow_dispatch` + polling hangs indefinitely on environment approval gates — the `gh run watch` command blocks and the run never progresses past the pending state.

### Release branch as break-glass
Each RC pipeline runs on a `release/<version>` branch, not `main`. If a called workflow has a defect mid-run:
1. Edit the broken workflow file on the `release/<version>` branch — fix the defect, and remove any jobs that already completed successfully in the previous run so they don't re-execute
2. Re-trigger `orchestrator-rc.yml` on that same branch with `previous_run_id` set to the failed run ID
3. The new run links back to the old run in its summary for traceability

This avoids needing `vars.*` break-glass flags or any special escape hatch logic in the YAML itself.

### Optional environments excluded from main orchestrators
Optional environments (e.g. canary regions) hold a concurrency slot indefinitely while waiting for approval. There is no per-job cancellation API in GitHub Actions — once a job is queued in a concurrency group, it blocks every subsequent pipeline from reaching that stage. Optional envs live only in `-optional` orchestrator variants that are triggered separately, not chained into the main flow.

### Pipeline supersession
Top-level `concurrency: group: orchestrator-rc, cancel-in-progress: false` ensures at most one pipeline runs and one queues at a time. A newer pipeline automatically supersedes any previously queued pipeline (GitHub cancels the old queued run).

The `approve-releasable-pipeline` job additionally runs a supersede loop: it cancels any older waiting runs and waits for any older in-progress runs to reach a gate before proceeding. This ensures the newest pipeline is always the one that moves forward.

### Deployment metadata via status description
GitHub auto-creates deployments when a job uses `environment:` but the payload is immutable. Version metadata (version, orchestrator run ID, server-demo run ID) is written into the `description` field of a subsequent deployment status after each environment completes. The `description` field supports up to 140 characters — sufficient for a compact JSON object:

```json
{"v":"2026.3.2","run":12345678901,"srv":12345678901}
```

Query pattern:
```bash
DEPLOYMENT_ID=$(gh api "repos/pixman20-org/approval-demo/deployments?environment=US-QA-DB&per_page=1" --jq '.[0].id')
gh api "repos/pixman20-org/approval-demo/deployments/$DEPLOYMENT_ID/statuses" \
  --jq '[.[] | select(.description != "")] | .[0].description'
```

### CI run tracing
`orchestrator-rc.yml` accepts a `ci_run_id` input (set by server-demo's build.yml). The `run-name` embeds this ID so the orchestrator run is findable by the server-demo run that triggered it. The summary links back to the triggering server-demo run.

When re-running after a failure, `previous_run_id` is passed so the new run's summary links to the run it continues from.

## GitHub Actions Limitations

- **No per-job cancellation**: Cannot cancel a specific queued job in another workflow run via API. Only full workflow run cancellation is available.
- **workflow_call snapshots at trigger time**: Called workflows are read from the ref at the moment of trigger. Edits to a called workflow only take effect if the orchestrator is re-triggered (hence the release branch pattern).
- **workflow_dispatch + environment gates deadlock**: `gh run watch` on a dispatched workflow blocks if that workflow is waiting on an environment approval — the approval requires a human, but the caller is also blocked waiting. Use `workflow_call` for any job that touches an environment gate.
- **run-name unreliable in workflow_call**: `inputs` context in `run-name` fields of called workflows does not render correctly. Remove `run-name` from reusable workflows.
- **One queued run per concurrency group**: If three pipelines start rapidly, only one can run and one can queue. The middle one is automatically cancelled by GitHub.
- **No interrupt-between-jobs**: There is no native way to cancel Pipeline A between its jobs when Pipeline B arrives. Pipeline A runs its current job to completion; B can only queue.
- **gh api 404 returns JSON to stdout**: `gh api` on a missing resource returns `{"message":"Not Found",...}` to stdout, not stderr. `--jq '.name'` returns the string `"null"` (truthy), not empty. Use `--jq '.name // empty'` and compare the result to the expected value.
- **Cascaded skips from needs**: If a job is skipped, all downstream jobs that list it in `needs:` are also skipped — even if those jobs have `if:` conditions. Override with `if: always()` combined with explicit result checks.
- **permissions: {} blocks workflow_call**: A `permissions: {}` at the workflow level blocks all permissions including those inherited from a workflow_call caller. Remove it from reusable workflows that need to inherit caller permissions.
- **workflow_call between repos not supported**: `uses:` with a reusable workflow only works within the same repository. Cross-repo reuse requires `workflow_dispatch` + polling (with the gate limitation above) or a composite action.
- **Deployment payload immutable**: The `payload` field set when a deployment is auto-created by GitHub cannot be updated. Use deployment status `description` as the metadata carrier instead.
