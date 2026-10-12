# Security policy

## Why this repository warrants a policy of its own

This action reads **your entire infrastructure tree** on your own runner and sends us what it
found. Infrastructure code carries account ids, bucket names, network topology and IAM policy, so
one property matters more than anything else here — and it is something you should be able to check
rather than take on trust:

- **Your source never leaves your runner.** The scanner Trivy produces a report that carries the
  offending source lines verbatim, at `Results[].Misconfigurations[].CauseMeta.Code.Lines[].Content`.
  That report is **not** forwarded. It is read on your machine, used to work out which Terraform
  resource a finding belongs to, and discarded. What is transmitted is rule id, severity, file path,
  line number and resource address.

  This is structural rather than a filter applied afterwards: the shape we transmit has no field for
  file contents, so there is nowhere for a line of your code to sit even by accident. The known
  limit: a finding's *title* and *remediation* come from the rule, not from your file, so they
  describe a class of defect rather than your code — if you ever see your own text in one, that is
  the bug this section is about.

Two more that follow from it:

- **A failed scan is never reported as a clean one.** A run that could not complete reports
  `complete: false`, and a partial scan never resolves anything absent from it. "No findings" and
  "we could not look" must never render the same way.
- **Nothing executes before it is verified.** The archive this action runs — our scanner, the Trivy
  binary and its policy bundle — is checked against a digest committed in `pins.env`, in the same
  commit you pinned, before anything in it is made executable. A re-tagged release or a replaced
  artefact FAILS the job instead of running, and does so regardless of `fail-on-error`: integrity is
  not an availability setting.

A defect in any of those is the most serious thing that can go wrong here, and we would rather hear
about it from you than from a customer.

## Verifying the first property yourself

You do not have to believe us. Run it with `dry-run: 'true'` and the action scans, builds the exact
payload it would send, and uploads nothing:

```yaml
- uses: Visiblaze/visiblaze-iac-scan@<commit-sha>
  with:
    tenant: your-tenant-id
    api-url: https://<your-visiblaze-endpoint>/v1/code/iac/findings
    dry-run: 'true'
```

Read the payload it prints. Every finding should carry a `path` and a `start_line` and nothing from
inside that file. If you ever see a line of your own Terraform there, that is a vulnerability in
this action and we want to know today.

## Reporting a vulnerability

Please report privately, not in a public issue.

Use **[GitHub private vulnerability reporting](https://github.com/Visiblaze/visiblaze-iac-scan/security/advisories/new)**
on this repository. If that is unavailable to you, email **security@visiblaze.com**.

Please include the action SHA you pinned, the scanner version from the job log, and — if the report
is about source leaking — the SHAPE of what leaked rather than the content itself. Do not send us a
live credential or a private resource identifier; if one leaked, rotate or redact it first.

## What is in scope

- Anything that causes file content or source text to be transmitted.
- Anything that causes an unverified or unpinned binary to execute.
- Anything that causes a failed or partial scan to be reported as complete.
- Anything that lets findings be sent to an origin not listed in `pins.env`.
- Anything that lets the scan path escape the checkout.

## What is not

- Findings you disagree with. A false positive is a detection-quality bug; open a normal issue.
- The absence of a check for a misconfiguration. Also a normal issue, and a welcome one.
