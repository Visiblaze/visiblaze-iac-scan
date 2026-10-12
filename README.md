# Visiblaze — IaC scan

Scans the infrastructure code in your repository, on your own runner.

```yaml
permissions:
  contents: read
  id-token: write        # mints a short-lived token; nothing is stored in your secrets

jobs:
  infrastructure:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Visiblaze/visiblaze-iac-scan@<sha>
        with:
          tenant: your-tenant-id
          api-url: https://…/v1/code/iac
          dry-run: "true"       # currently the only mode that completes — see Status
```

## What leaves your runner

Findings and locations. **Never your code.**

That is enforced, not promised. Trivy's own report carries the offending source verbatim, so this
action does not forward it. The scanner normalises on your machine into a shape that has no field
for file contents:

| Sent | Not sent |
|---|---|
| rule id, severity, title | the line that triggered it |
| path and line number | the file's contents |
| Terraform resource address | variable values, secrets, state |
| how many times a module is instantiated | anything outside the finding |

Your source is read on the runner, used to work out which resource a finding belongs to, and left
there. If our storage were breached tomorrow, what an attacker would hold is a list of rule ids
and line numbers — not your infrastructure.

## What it reads

Terraform (`.tf`, `.tf.json`), Dockerfiles, and workflow YAML under `.github/`. Vendored and
generated trees are skipped: `.terraform`, `vendor`, `node_modules`, `testdata`, fixtures and test
directories. Symlinks are not followed, so the scan cannot be pointed out of your checkout.

## Reading the result

`complete: false` **with zero findings is not a clean repository.** It means the scan did not read
everything it was pointed at — usually a timeout — and the step summary says so. A consumer that
treats the two alike will report an all-clear for a scan that never finished.

`fail-on` is your policy and fails the job on its own terms. `fail-on-error` governs only whether
*our* failures break your build, and defaults to `false`: a scanner outage should not be the reason
your pipeline goes red.

## Timing

Budget generously. Measured on the repository this scanner was built in, a ~1,100-file
infrastructure tree takes **about 5m35s**, and a 6-minute budget timed out on one run of two. The
default is 14 minutes for that reason.

## Supply chain

One artefact, one digest, one URL. The scanner, the Trivy engine and its policy bundle ship
together and are verified against a digest committed in `pins.env` before anything is executed. A
substituted artefact fails the check rather than running. Nothing is fetched from a registry at
scan time, so a registry outage cannot silently turn a scan into a no-op.

## Status

**Not yet shippable.** Two things are missing and both are ours, not yours:

- there is no released `visiblaze-iac` binary, so `pins.env` carries no digest and the action
  refuses to run rather than fetch something unverified;
- there is no ingest route for infrastructure findings, so a non-dry run has nowhere to post.

`dry-run: true` works today and is genuinely useful: it scans, builds the payload, validates it,
and prints what would be sent.
