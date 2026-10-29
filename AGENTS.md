# Zkuiz agent instructions

## Product and stack

- Build a practice platform for zero-knowledge arithmetic circuits. LeetCode is the primary interaction reference; Zkuiz has its own visual identity.
- Use Next.js, Bun, and shadcn/ui. The application and test harness have not been scaffolded yet; do not claim existing package commands or CI gates.
- Use composable components. Keep UI primitives in `components/ui`, domain components in their feature, and API/job orchestration outside presentation components. Extract abstractions when there is a real shared responsibility.
- Write public documentation for human readers. Keep README and contribution guidance concise and professional; put agent skills, MCP setup, and session-specific workflow details in this file rather than public onboarding. Do not describe planned features as available.
- For UI work, apply the installed `mba-design-rules` and the relevant MBA build skill. Use `quiet-signal-design` for the default personal engineering style unless an explicit brief overrides it.

## Branches and review

- Follow [CONTRIBUTING.md](CONTRIBUTING.md). Create a focused branch before changing tracked files. Use one reviewable purpose per PR.
- Start independent changes from current `main`. Stack PRs only when there is a real dependency; state the parent PR and base branch.
- Never push implementation directly to `main`, merge a PR, or enable auto-merge without explicit user instruction. Leave completed changes for user review.
- Commit and stage only the intended files. Preserve other work in the shared checkout.
- Use Conventional Commit titles. Prefer squash merge and delete merged branches after GitHub confirms the merge.
- In T3 Code, register every PR with `link_pull_request` immediately after creation or starting work, and reconcile `list_thread_pull_requests` before handoff.

## TDD for each behavior change

1. State the acceptance behavior and add a focused test first.
2. Run it and observe an expected failure caused by the missing behavior. A setup/import/dependency failure does not establish Red.
3. Make the smallest implementation that turns the test Green.
4. Refactor while keeping the relevant tests Green.
5. Run the project's scoped checks and record the actual Red/Green commands and results in the PR.

- Tests should cover behavior and meaningful boundaries, not duplicate implementation details or fragile markup.
- Apply TDD to features and bug fixes. For documentation, metadata, or a purely visual adjustment, explain why executable regression coverage is not applicable and perform the relevant checks instead.
- Discover commands from the project's own scripts and configuration. The first application-code PR must establish a runnable test harness and relevant CI checks.
- If tests cannot run, report the blocker and keep the PR draft. Do not claim TDD or verification passed.

## Verifier for every feature

- Apply the installed `freebuff-verifier` workflow to every feature and behavior-changing fix before handoff. This is a recurring requirement, not a one-time setup check.
- Inspect the final diff for correctness, edge cases, scope, and unintended edits. Run focused tests and static checks for the changed behavior; exercise changed UI interactions when practical.
- Fix failures introduced by the change and rerun affected checks. Report commands, outcomes, and remaining limits in the PR.
- Choose additional reviewers, browser checks, and broader suites according to risk. A full suite or separate reviewer is not automatically required for every change.
- If the skill is unavailable, disclose that and perform the same workflow manually; never claim the skill ran.

## Codebase Memory MCP

- Use the installed `codebase-memory` skill for structural code exploration. Match the current checkout to its indexed project; initialize or refresh the graph before relying on it.
- Use graph search, traces, and snippets first. Check coverage for every file relied on. Read reported missed ranges directly and qualify incomplete or stale coverage.
- Keep indexes, local MCP configuration, and machine-specific ignore rules local. The default index cache is outside the repository. Use `.git/info/exclude` for local Git exclusions and `.cbmignore` for local indexing exclusions; do not commit either generated state or local configuration.
- Git exclusion and index exclusion are different. Verify both explicitly. Do not assume `.git/info/exclude` alone controls the indexer.
- Do not run a global MCP installer or change account-wide settings merely to index this repository when the server is already available.

## Judge boundary

- Keep hidden tests, secrets, and authoritative verdict policy on the server. Browser execution is not an official acceptance decision.
- Treat user code as untrusted. Establish and verify worker isolation before accepting arbitrary execution.
- Keep job snapshots separate from mutable editor drafts. Preserve the distinction between test results, formal verification, and inconclusive solver outcomes.
- Use current primary documentation or Context7 when library or API details matter. Keep durable project decisions sourced and dated.
