# DidWork verify action

Your CI said the deploy worked. Did it?

This action runs the claims in your repo's `.didwork.yml` against the systems
that can prove them — Stripe, GitHub, your own health endpoint — and fails the
job when a required outcome is not true. Because it fails the job, you can make
it a required status check and stop merges that rest on a self-report.

```yaml
name: Verify outcomes
on: pull_request

permissions:
  contents: read
  pull-requests: write   # only needed for the PR comment

jobs:
  didwork:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: didworksh/verify-action@v1
        with:
          api-key: ${{ secrets.DIDWORK_API_KEY }}
```

## The gate file

```yaml
version: 1
claims:
  - name: Production health
    type: http.ok
    expected:
      url: https://example.com/health

  - name: Release published
    type: github.release_published
    expected:
      repository: acme/app
      tag: v1.4.0

  - name: Canary (reported, cannot block)
    type: http.ok
    required: false
    expected:
      url: https://canary.example.com/health
```

Every claim is required unless it says `required: false`. A non-required claim
still runs and still appears in the report; it just cannot fail the job.

## What passes

A required claim passes the gate only when its verdict is `verified`.

| Verdict | Meaning | Required claim |
| --- | --- | --- |
| `verified` | the outcome is true | passes |
| `failed` | the outcome is not true | **blocks** |
| `unknown` | could not be established | **blocks** |
| `error` | the check could not be run | **blocks** |

`unknown` blocking is deliberate. A check that could not reach its evidence has
not shown anything to be true, and a green merge button backed by nothing is the
exact failure this is here to prevent.

## Permissions

The comment is written with **your** workflow's `GITHUB_TOKEN`, under whatever
scope you granted it. DidWork holds read-only credentials to the systems it
verifies and has no write access to your repository — it can observe an outcome,
never cause one. Drop `pull-requests: write` and set `comment: false` if you want
the gate without the comment; the job's pass/fail is unchanged.

## Making it a required check

Settings → Branches → branch protection rule → **Require status checks to pass**,
then select the job (`didwork` in the example above).

## Without an API key

The gate still runs, but only `http.ok` claims verify, nothing is stored, and
there are no receipts. Provider-backed claims (Stripe, GitHub, Linear, Sentry, …)
need a key and a connected provider: https://didwork.sh/console

## Reviewed outcome contracts

Version 1.1.0 adds version 2 contracts with stored aggregate runs. Configure
`contract: true`, `trusted-ref` (the full SHA of the reviewed contract), and
`revision` (the full deployed SHA). This mode needs an API key. The CLI is pinned
to 0.3.0 by default.

The protected workflow must supply the trusted SHA; do not take it from PR code.
Fetch that Git object before running the action. Use per-PR/environment workflow
concurrency with `cancel-in-progress: true` and pin the Action to a reviewed
commit. Required unknown or failed checks block the job. `comment: true` shares
a redacted public summary, so review the contract's optional public descriptions.

See [the contract guide](https://didwork.sh/docs#contracts)
and [deployment example](https://github.com/didworksh/verify-action/blob/main/examples/deployment.didwork.yml).
