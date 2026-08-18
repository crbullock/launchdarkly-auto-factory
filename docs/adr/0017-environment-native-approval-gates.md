# ADR 0017 — Environment-native approval gates (phase-split jobs)

**Status:** proposed

**Context.** Phase 1 runs the whole agent chain (research → planner → manifest-steward →
flag-implementer → metrics → tests) in a single GitHub Actions run, and pauses for human
approval via the LD approval-policy flags (ADR 0008). Because an Actions run is ephemeral
and stateless, "pause" is implemented as *exit the run, and re-run later when a human
signals*; the signal is a PR label (`af-approve:<step>`), and the `labeled` event re-fires
the workflow, which re-walks the chain and proceeds past the now-open gate.

Live testing on `flexapp/flex2-orchestration` exposed two structural problems with that
resume model:

1. **`[skip ci]` on the PR HEAD silences the resume event.** The agents' own commits carry
   `[skip ci]` as a loop guard. GitHub suppresses *all* `pull_request` activity (including
   `labeled`) when the HEAD commit contains `[skip ci]`. So once the manifest is committed,
   adding `af-approve:*` produces no run — the chain cannot advance. Removing `[skip ci]`
   fixes the label but then every auto-factory commit self-triggers a fresh
   `pull_request: synchronize` run (empirically confirmed — GITHUB_TOKEN anti-recursion did
   *not* suppress these), producing a **cascade** in multi-commit stages: each commit
   re-runs the *entire* LLM chain. Neither state is acceptable; we currently keep `[skip ci]`
   and accept that label-driven resume does not work (stages 1–2 only).

2. **Resume re-runs the whole chain.** Even when a re-run fires, it re-executes research and
   planning from the top (real token cost, possible non-determinism) to advance a single
   step, because there is no persistent process holding the paused chain.

A short-term patch is to resume via an `issue_comment` trigger (`/af-approve`), which is not
a `pull_request` event and so is not silenced by `[skip ci]`. That keeps the re-run model
(and its per-resume chain re-execution) but unblocks resume. This ADR proposes the
structural alternative that removes the re-run model entirely.

**Decision.** Split the action into **phase-scoped jobs** gated by **GitHub Environments
with required reviewers**. A job that targets a protected environment pauses in a `waiting`
state until a designated reviewer approves in the Actions UI — natively, with no re-run and
no runner minutes consumed while waiting — then the *same* job continues.

Job graph (one workflow run, pausing between jobs):

```
plan  ──▶  [env: auto-factory-build]  build  ──▶  [env: auto-factory-implement]  implement
```

- **`plan`** — always runs, dry-run. Research + planner only; writes the manifest to the
  working tree (never committed) and posts the concise plan-preview comment. Uploads the
  manifest as an artifact.
- **`build`** — gated by the `auto-factory-build` environment. On approval, commits the plan
  manifest to the PR branch.
- **`implement`** — gated by the `auto-factory-implement` environment. On approval, creates
  the flag (dark), wires the code, adds metrics + tests, commits them.

The manifest rides `plan → build → implement` as an artifact so downstream jobs consume the
plan instead of re-deriving it. This requires the action to expose a **`phase`** input
(`plan | build | implement`) so each job executes only its slice of the chain.

Workflow sketch:

```yaml
name: LaunchDarkly Auto-Factory
on:
  pull_request:
    types: [opened, synchronize, reopened]   # no `labeled`
permissions: { contents: write, pull-requests: write, checks: write }

jobs:
  plan:
    runs-on: ubuntu-latest
    if: <trusted same-repo PR gate>
    steps:
      - uses: actions/checkout@v4
        with: { ref: ${{ github.event.pull_request.head.sha }}, fetch-depth: 0 }
      - uses: launchdarkly-labs/launchdarkly-auto-factory/packages/phase1-resource-factory@<sha>
        with: { phase: plan, ld_sdk_key: ${{ secrets.LD_SDK_KEY }}, ... }
      - uses: actions/upload-artifact@v4
        with: { name: release-manifest, path: .release-flags/ }

  build:
    needs: plan
    runs-on: ubuntu-latest
    environment: auto-factory-build          # required-reviewers protection rule
    steps:
      - uses: actions/checkout@v4
        with: { ref: ${{ github.event.pull_request.head.ref }}, fetch-depth: 0 }
      - uses: actions/download-artifact@v4
        with: { name: release-manifest, path: .release-flags/ }
      - uses: launchdarkly-labs/launchdarkly-auto-factory/packages/phase1-resource-factory@<sha>
        with: { phase: build, ... }

  implement:
    needs: build
    runs-on: ubuntu-latest
    environment: auto-factory-implement      # second required-reviewers gate
    steps:
      - uses: actions/checkout@v4
        with: { ref: ${{ github.event.pull_request.head.ref }}, fetch-depth: 0 }
      - uses: actions/download-artifact@v4
        with: { name: release-manifest, path: .release-flags/ }
      - uses: launchdarkly-labs/launchdarkly-auto-factory/packages/phase1-resource-factory@<sha>
        with: { phase: implement, enable_flag_creation: "true", enable_code_changes: "true", ... }
```

**Consequences.**

Wins:
- **No re-run to resume.** The pause is a real job-level wait; the chain is not re-executed,
  so no wasted LLM tokens on resume. This is the core reason to do it.
- **`[skip ci]` and the label/`issue_comment` mechanics disappear.** Approvals are native
  "Review deployments → Approve" clicks, surfaced on the PR's checks. No event-suppression
  edge cases.
- **Safe by construction.** `plan` is dry-run; `build`/`implement` cannot start until a human
  approves the environment. The "never touch a PR unattended" requirement holds without
  relying on a remotely-flippable flag.

Costs / open questions:
- **Action rework: `phase` entrypoints.** The action must run only research+plan, only the
  manifest commit, or only implement. Without this, each gated job re-runs the whole chain
  and the main benefit is lost. This is the bulk of the work and is an action change (not
  expressible from the workflow side).
- **State handoff via artifacts.** `plan → build → implement` pass the manifest as an
  artifact; the action's `build`/`implement` phases must accept an input manifest rather than
  re-deriving it.
- **Per-repo setup in GitHub settings.** Each consuming repo must create the
  `auto-factory-build` and `auto-factory-implement` environments and configure required
  reviewers. A drop-in workflow file alone is no longer sufficient (contrast ADR 0003).
- **Approval UX moves off the PR** into the Actions "Review deployments" panel (linked from
  PR checks) — less PR-native than a label/comment.
- **Partially supersedes ADR 0008.** The GitHub-native gate replaces the LD
  `auto-factory-approval-mode` / `auto-factory-approval-gates` flags for these two gates.
  **Risk-*conditional* gating** (gate only when `risk_score ≥ threshold`) is awkward:
  environment protection is static, so low-risk auto-approval would require the `plan` job to
  conditionally skip the gated jobs via an `if:` on a planner-emitted output, rather than the
  flags deciding. The environments model is cleanest for "always require approval."

Relationship to other ADRs: builds on ADR 0003 (PR-open trigger) and reshapes the approval
mechanism of ADR 0008 (approval policy compiles into gates). Does not change the manifest /
release-intent contract (ADR 0009) or the release path (ADR 0002).

**Recommendation.** Ship the `issue_comment` resume first (small, keeps the current model and
unblocks stage-3 resume today). Adopt this ADR when the action gains `phase` support; treat
it as the target architecture for approval gating.
