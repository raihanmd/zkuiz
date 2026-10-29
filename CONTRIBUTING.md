# Contributing to Zkuiz

Zkuiz is in early development. Useful contributions include circuit exercises, adversarial test cases, verification research, and documentation improvements.

For a substantial change, open an issue first to discuss the problem and proposed approach. Small corrections can go directly to a pull request. Please follow the [code of conduct](CODE_OF_CONDUCT.md).

## Getting started

Fork the repository, clone your fork, and create a branch for your change:

```sh
git clone https://github.com/YOUR-USERNAME/zkuiz.git
cd zkuiz
git switch -c feat/describe-your-change
```

The application, test harness, and CI have not been added yet. There are no installation or development-server commands to run. The planned stack is Next.js, Bun, and shadcn/ui; setup instructions will accompany the application code.

## Scope

Keep each pull request focused on one problem. Include the tests and documentation needed to understand the change, and separate unrelated improvements into their own PRs.

For circuit exercises or checking methods, describe the input domain, expected constraints, and cases that should be rejected. Distinguish test coverage from formal guarantees, and cite sources for research claims.

When application work begins, compose UI from small components with clear responsibilities. Keep presentation separate from API requests and job orchestration.

## Tests and verification

Use test-driven development for features and bug fixes:

1. Write a test that captures the expected behavior and confirm it fails for the intended reason.
2. Implement the change and confirm the test passes.
3. Refactor as needed, then rerun the relevant checks.

Include the commands and results in the PR. Review the final diff for edge cases and unintended changes. For UI changes, describe the interactions you checked.

Documentation-only changes do not require an executable regression test. Check links, examples, and factual accuracy instead. If a required check cannot run, explain the limitation and keep the PR in draft.

The first application-code contribution must establish a runnable test harness and CI checks. Subsequent contributions should use the commands documented by the project.

## Pull requests

- Branch from the current `main`. Use a descriptive name such as `feat/range-check-exercise`, `fix/testcase-parser`, or `docs/circuit-verification`.
- Explain the problem, the change, and how you verified it. Link related issues and note any dependent PRs.
- Use a Conventional Commit title, such as `docs: clarify circuit verification limits`.
- Keep incomplete work in draft. Address review feedback before requesting another review.

Maintainers review and squash-merge changes into `main`. Do not rewrite a branch under active review without coordinating with the reviewers.

## Security reports

Do not disclose vulnerabilities in public issues. Follow [SECURITY.md](SECURITY.md) to submit a private report.

## License

Contributions are accepted under the project's [MIT license](LICENSE).
