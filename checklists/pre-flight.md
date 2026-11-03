# Pre-flight checklist

- [ ] Finding is in target/library code, not the harness.
- [ ] Reproduced on an unmodified upstream commit/tag.
- [ ] `git status` is clean at the tested commit.
- [ ] Exact commit and release/version recorded.
- [ ] Root cause traced to a specific sink with file:line.
- [ ] POC minimized.
- [ ] POC deterministic across at least 3 runs.
- [ ] Benign control input exits cleanly.
- [ ] Non-sanitized build behavior checked.
- [ ] Duplicate search done (project issues/PRs, CVE, OSS-Fuzz, ARVO).
- [ ] Severity and CWE chosen from demonstrated effect.
- [ ] The same fix would close this issue and no other root cause is mixed in.
