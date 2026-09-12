# gh-common-actions
Github workflows/actions library

## run-robotframework.yml

Reusable workflow that runs a target's Robot Framework suite
(`test_command`, default `make run`) and uploads the resulting `results/`
directory as a build artifact. Nothing else — no bootstrap, no lint, no
static checks; runner images are assumed to already have Python/robot
tooling installed. If a target needs setup first, fold it into
`test_command`.

This workflow is entirely Allure-agnostic — it has no `enable_allure` input
and no Allure output. If a target wants an Allure report, that's entirely
between the caller's own job and `publish-allure.yml`; see the example
under `publish-allure.yml` below for how a caller wires the listener and
artifact upload itself, independent of this workflow.

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
