# gh-common-actions
Github workflows/actions library

## run-robotframework.yml

Reusable workflow that runs a target's Robot Framework suite following the
`bootstrap -> lint -> check -> run` Makefile convention (see
`oguz-labs/old-scout/tests/Makefile`), wires the `allure_robotframework`
listener into the run automatically, and uploads both the native RF
artifacts (`output.xml`/`log.html`/`report.html`) and the generated
`allure-results` directory as build artifacts. It does not publish to the
shared Allure server — chain its `allure_results_dir` output into
`publish-allure.yml` from the calling workflow.

```yaml
jobs:
  test:
    uses: oguz-labs/gh-common-actions/.github/workflows/run-robotframework.yml@main
    with:
      working_directory: tests
      sut_url: https://legacy-scout.internal

  publish:
    needs: test
    uses: oguz-labs/gh-common-actions/.github/workflows/publish-allure.yml@main
    with:
      project_name: legacy-scout
      report_source_type: robotframework
      results_dir: ${{ needs.test.outputs.allure_results_dir }}
```

## publish-allure.yml

Reusable workflow that pushes an already-generated Allure results directory
to the shared `allure-server` (namespace `allure-reports`) and renders the
report. It does not run your tests or convert report formats — your job
must produce native Allure results first (Robot Framework via the
`allure_robotframework` listener, Playwright via the `allure-playwright`
reporter), then call this workflow.

```yaml
jobs:
  test:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: robot --listener allure_robotframework:allure-results suites/

  publish:
    needs: test
    uses: oguz-labs/gh-common-actions/.github/workflows/publish-allure.yml@main
    with:
      project_name: legacy-scout
      report_source_type: robotframework
      results_dir: allure-results
```

See `oguz-labs/elastic-automation/docs/REGRESSION_REPLAY.md` §7 and
`oguz-labs/oguz-lab-infra/docs/adr/ADR-009-Allure-Report-Upload-Guide.md`
for the conventions this workflow enforces.
