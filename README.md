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

## run-crawler.yml

Reusable workflow that runs elastic-automation's autonomous crawler
(`observer.crawler`) against a target's SUT and produces this run's
`manifest.yaml` + `elastic-pom.yaml` as a single build artifact. Nothing
else — no diff, no policy gate, no checkin.

This workflow has **zero knowledge of the target's own repo layout or
commit conventions**. It never writes into `bindings_dir`/`artifacts_dir`
inside the checkout, and it never commits or pushes anything — it writes
both output files to a workflow-local scratch directory and uploads them
as one artifact (`artifact_name`, default `crawler-output`). What the
caller does with that artifact — unpacking it into its own `bindings_dir`,
committing `elastic-pom.yaml`, opening a PR — is entirely the target
project's own job and its own engineers' call, never this workflow's.

```yaml
jobs:
  crawl:
    uses: oguz-labs/gh-common-actions/.github/workflows/run-crawler.yml@main
    with:
      sut_url: https://legacy-scout.internal
      manifest_id: ui:legacy-scout:spec-1
      version: v0.2.0
```

Comparing the crawl's output against a promoted baseline is a separate
concern — see `replay.yml` below.

## replay.yml

Reusable workflow that runs elastic-automation's `observer.replay_cli`
against a downloaded `crawl` artifact (from `run-crawler.yml` or any job
that produces the same two files) and a target's promoted baseline
directory, applies the Process 2 CI policy
(`oguz-labs/elastic-automation/docs/REGRESSION_REPLAY.md` §7), and posts
the Markdown report as the job summary. Nothing else — no crawl, no
checkin, no promotion.

`baseline_dir` is read from the checkout as-is (it's the target's own
committed, durable state) — this workflow never writes to it.

```yaml
jobs:
  crawl:
    uses: oguz-labs/gh-common-actions/.github/workflows/run-crawler.yml@main
    with:
      sut_url: https://legacy-scout.internal
      manifest_id: ui:legacy-scout:spec-1
      version: v0.2.0

  replay:
    needs: crawl
    uses: oguz-labs/gh-common-actions/.github/workflows/replay.yml@main
    with:
      artifact_name: ${{ needs.crawl.outputs.artifact_name }}
      manifest_id: ui:legacy-scout:spec-1
      baseline_dir: tests/baseline
      version: v0.2.0
```

The job's exit code mirrors `replay_cli`'s policy verdict (0
pass/warn, 1 require_review, 2 block, 3 no baseline found) — a failing
`replay` job is the CI gate. Promoting a new baseline is a separate,
human/TTL-authorized step, not something either workflow does.
