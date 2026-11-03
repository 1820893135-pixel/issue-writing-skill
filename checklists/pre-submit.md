# Pre-submit checklist

- [ ] Title names the function and trigger, not just "crash".
- [ ] Affected version/commit is exact.
- [ ] Build command was tested from a clean checkout.
- [ ] Driver is minimal and uses a public or clearly documented entry point.
- [ ] POC size, SHA-256, hex recipe, and layout table are present.
- [ ] Full sanitizer/abort stack and exit code are present.
- [ ] Non-sanitized behavior is stated.
- [ ] Impact separates demonstrated from theoretical.
- [ ] Suggested fix names the invariant and includes a regression test.
- [ ] No `/root`, `/tmp`, `/home`, `C:\Users`, hostnames, or tokens.
- [ ] Links resolve.
- [ ] Correct GitHub account is active (`gh auth status`).
- [ ] Project security channel checked before public posting.
