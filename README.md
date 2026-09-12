# gh-common-actions
Github workflows/actions library

## run-robotframework.yml

Reusable workflow that runs `make bootstrap && make run` for a target's
Robot Framework suite and uploads the whole `results/` directory as a build
artifact. The suite run itself has no Allure awareness by default — flip
`enable_allure: true` to wire the `allure_robotframework` listener into the
run (appended to any `ROBOT_OPTIONS` the caller already set, never
overwriting it) and get the resulting `allure-results` directory uploaded
too, for chaining into `publish-allure.yml`. `lint`/`check` are off by
default and only worth turning on for a target whose Makefile actually
defines them — `check` in particular (a static test-intent trace) is an
elastic-automation convention, not something every target has.

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
