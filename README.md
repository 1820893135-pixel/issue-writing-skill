# Issue Writing Skill

A practical playbook for turning a fuzzing or verification result into a
reproducible vulnerability issue, and then into a CVE request.

The core skill is [SKILL.md](SKILL.md). It is written so it can be used by a
human researcher or loaded as a Claude Code skill.

## What is here

| Path | Purpose |
|---|---|
| `SKILL.md` | The full skill: principles, workflow, issue structure, CVE flow, checklists, and lessons. |
| `templates/github-issue.md` | Generic GitHub issue body. |
| `templates/cve-request.md` | CVE request fields and description template. |
| `checklists/pre-flight.md` | What to verify before writing. |
| `checklists/pre-submit.md` | What to verify before publishing. |

## Quick start

1. Read the non-negotiable rules in `SKILL.md`.
2. Reproduce on a clean checkout of the exact upstream commit/tag.
3. Build a minimal public-API driver with ASan/UBSan.
4. Minimize the POC; record size, SHA-256, hex, and layout.
5. Capture the full stack, exit code, and a benign control.
6. Trace the root cause from input to sink.
7. Search for duplicates and known/ARVO samples.
8. Write the issue with `templates/github-issue.md` or the project's form.
9. Sanitize paths and secrets.
10. Post with the intended GitHub account; follow up with evidence.

## Use as a Claude Code skill

Copy `SKILL.md` to:

```text
<project>/.claude/skills/vulnerability-issue-writing/SKILL.md
```

or keep this repository as a reference playbook.

## The short version

- One root cause per issue.
- Exact version and clean source.
- A stranger must be able to reproduce it.
- POC + build + stack + exit code are mandatory.
- Do not publish internal paths, credentials, or private data.
- Do not overclaim impact.
- Check duplicates and known benchmarks first.
- Use the project's security channel and template.
- For CVE requests, use the current MITRE form and never publish a CAN as a CVE.

## Current CVE request entry point

https://mitre.github.io/mitre-cve-roles/cve-id-request/

See `SKILL.md` section 7 for the full flow and the CAN-vs-CVE distinction.
