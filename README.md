# gh-common-actions
Github workflows/actions library

## run-robotframework.yml

Reusable workflow for a target's install/lint/check/test steps and results
upload. Nothing assumes `make` — every step is a plain command string
(`bootstrap_command`/`lint_command`/`check_command`/`test_command`); the
defaults match `old-scout/tests/Makefile` for convenience only, since
elastic-automation's onboarding convention doesn't guarantee any target has
a Makefile at all. `lint_command`/`check_command` are empty (skipped) by
default — `check` in particular (a static test-intent trace) is an
elastic-automation convention, not something every target has an
equivalent of. Flip `enable_allure: true` to export `ROBOT_OPTIONS` with
the `allure_robotframework` listener before `test_command` runs (appended
to any `ROBOT_OPTIONS` the caller already set, never overwriting it — and
only effective if `test_command`'s own tooling reads that env var) and get
the resulting `allure-results` directory uploaded too, for chaining into
`publish-allure.yml`.

Jobs never share a filesystem in GitHub Actions, so chaining into
`publish-allure.yml` goes through an actual artifact — `enable_allure`
uploads one named `allure-results`, and `publish-allure.yml`'s
`artifact_name` input downloads it back down in the next job:

```yaml
jobs:
  test:
    uses: oguz-labs/gh-common-actions/.github/workflows/run-robotframework.yml@main
    with:
      working_directory: tests
      sut_url: https://legacy-scout.internal
      enable_allure: true

  publish:
    needs: test
    if: needs.test.outputs.allure_results_dir != ''
    uses: oguz-labs/gh-common-actions/.github/workflows/publish-allure.yml@main
    with:
      project_name: legacy-scout
      report_source_type: robotframework
      artifact_name: allure-results
```

## publish-allure.yml

Reusable workflow that pushes an already-generated Allure results directory
to the shared `allure-server` (namespace `allure-reports`) and renders the
report. It does not run your tests or convert report formats — your job
must produce native Allure results first (Robot Framework via the
`allure_robotframework` listener, Playwright via the `allure-playwright`
reporter), then call this workflow.

If your test step and this workflow run in the same job, `results_dir` just
needs to point at the directory on disk. If they're separate jobs (the
usual case — this workflow needs its own `runs-on: self-hosted` to reach
the in-cluster server), upload the results as an artifact in the test job
and pass its name via `artifact_name` instead — jobs don't share a
filesystem, so `results_dir` alone won't carry files across a job boundary:

```yaml
jobs:
  test:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: robot --listener allure_robotframework:allure-results suites/
      - uses: actions/upload-artifact@v4
        with:
          name: allure-results
          path: allure-results

  publish:
    needs: test
    uses: oguz-labs/gh-common-actions/.github/workflows/publish-allure.yml@main
    with:
      project_name: legacy-scout
      report_source_type: robotframework
      artifact_name: allure-results
```

See `oguz-labs/elastic-automation/docs/REGRESSION_REPLAY.md` §7 and
`oguz-labs/oguz-lab-infra/docs/adr/ADR-009-Allure-Report-Upload-Guide.md`
for the conventions this workflow enforces.
