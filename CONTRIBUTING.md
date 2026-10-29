# Development workflow

Zkuiz uses Next.js, Bun, and shadcn/ui. The current repository contains project setup; application code, test commands, and CI will be established by the first implementation PR.

## Branch model

`main` is the stable integration branch. Each change uses a short-lived branch with one purpose:

- `feat/<behavior>` for a feature.
- `fix/<bug>` for a correction.
- `refactor/<scope>`, `test/<scope>`, `docs/<topic>`, or `chore/<setup>` for other changes.

Start independent work from an up-to-date `main` in a clean checkout. Do not discard or stash another contributor's work to make the checkout clean.

```sh
git switch main
git pull --ff-only
git switch -c feat/<behavior>
```

Keep PRs focused enough to review in one sitting. Split unrelated setup, research, and product behavior. Do not use an arbitrary line limit that separates a change from the tests it needs.

Use stacked PRs only for dependent changes. State the parent and base in each description, keep each layer independently understandable, and retarget dependent PRs after their parent merges. Independent branches should both target `main`.

## Test-first implementation

For every feature or bug fix:

1. Define the observable acceptance conditions.
2. Add the smallest meaningful failing test and run it. Verify the failure is due to the missing behavior.
3. Implement enough to pass, then refactor with tests still passing.
4. Run focused project checks and inspect the final diff.
5. Include the actual Red and Green commands and their results in the PR.

Choose the test level according to the behavior: unit tests for isolated rules, integration tests for boundaries, and interaction tests for user flows. Do not invent a test to satisfy a checkbox or claim an unobserved failure.

Documentation, metadata, and purely visual changes may use targeted non-test checks with a stated reason. Before the first application-code PR is ready, establish the project test harness and relevant CI checks. Test and CI commands are intentionally not prescribed before those tools exist.

## Feature verification

Apply the installed `freebuff-verifier` workflow for every feature and behavior-changing fix. Inspect correctness, edge cases, and unintended edits; run the checks that cover the changed behavior; exercise user-facing changes when practical. Resolve introduced failures before handoff.

Use additional reviewers or broader checks when the risk warrants them. In particular, authentication, untrusted-code execution, worker isolation, and judge correctness need evidence at the affected boundary. A separate reviewer and full suite are not mandatory for every change.

If the skill is unavailable, state that and perform the equivalent checks manually. If checks cannot run, document the blocker and keep the PR draft.

## Pull requests and merge

- Open a PR for each focused change. Use a Conventional Commit title such as `feat: run a circuit against example inputs`.
- Describe the problem, resulting behavior, validation evidence, and material limits. Complete the PR template honestly.
- Keep draft PRs for incomplete work or blocked verification. Otherwise mark the change ready for user review.
- Agents do not merge or enable auto-merge without explicit user instruction.
- Use squash merge. GitHub deletes the merged branch automatically. Preserve review conversations and resolve them before merge.
- Do not rewrite a published branch while someone is reviewing it without coordinating first. Never force-push `main`.

Repository settings require PRs, resolved conversations, and linear history on `main`, and prohibit force-pushes and branch deletion there. Required approvals are currently zero because agent PRs use the repository owner's GitHub identity, and GitHub does not allow authors to approve their own PR. The owner reviews and merges manually; this setup does not pretend to enforce an independent approval.

No required CI status checks are configured yet. Add the actual checks with the first test harness rather than referencing nonexistent jobs.

## Local Codebase Memory setup

Codebase Memory MCP is available in the current development environment. The installed CLI supports indexing an absolute path:

```sh
codebase-memory-mcp cli index_repository --repo-path /absolute/path/to/zkuiz
```

The default database cache is `~/.cache/codebase-memory-mcp/`, outside Git. Do not relocate it into the project or change account-wide cache settings merely for this repository.

For local artifacts and machine-specific configuration, add rules to `.git/info/exclude`. This file belongs to the local Git metadata and is never committed. For example:

```gitignore
/.cbmignore
/.mcp.json
/.codebase-memory/
/.codebase-memory-mcp/
/.playwright-mcp/
```

`.cbmignore` controls indexing exclusions and can itself be excluded locally. Put generated folders such as `.playwright-mcp/`, `node_modules/`, `.next/`, and `coverage/` there when needed. A Git exclusion does not necessarily imply an index exclusion.

Confirm a local artifact is ignored with `git check-ignore -v <path>` and confirm it is absent from `git ls-files`. Check `index_status` and `check_index_coverage` for index state and coverage. A documentation-only repository does not yet provide a meaningful application call graph.

Sources: [Codebase Memory documentation](https://github.com/DeusData/codebase-memory-mcp), [GitHub local ignore rules](https://docs.github.com/en/get-started/git-basics/ignoring-files).
