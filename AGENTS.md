## Agentic Workflows

After modifying any `.md` workflow file under `.github/workflows/`, always
recompile and commit the generated workflow files with the source change:

```bash
gh aw compile
apm compile
```

For Goal issues, keep the completion contract evidence-based. A goal is complete
only when the issue's stated verification evidence supports it.

## Known Flaky Tests

These fail intermittently for reasons unrelated to most changes. If one fails,
check its issue before investigating. When the issue closes, remove its line
here.

- `action_cross_node` in `scripts/test_mkn.py`: client fails with "Import fetch
  failed for service 'B'". [#204](https://github.com/meerkat-lang/meerkat/issues/204)
- `mkn_client_timeout_slow` in `scripts/test_mkn.py`: runs no Meerkat code,
  only a 2-second `sleep`, yet sometimes times out.
  [#205](https://github.com/meerkat-lang/meerkat/issues/205)

Run `cargo build -p meerkat` before `scripts/test_mkn.py`. It launches nodes
with `cargo run`, so if the binary is stale the first node spends its startup
time compiling and fails with "Timeout waiting for node ... to initialize".
Pre-commit hooks build with `--all-features`, so this is common right after a
commit.

List all known flaky tests with `gh issue list --label flaky`.
