---
name: vulnerability-issue-writing
description: Write reproducible, evidence-based vulnerability issues for open-source projects, adapt them to project-specific GitHub forms, and prepare CVE requests. Use when a fuzzing or verification result needs to become a public issue or a CVE request.
---

# Vulnerability Issue Writing Skill

This skill turns a confirmed crash or sanitizer finding into an issue that a
maintainer can reproduce without access to the researcher's machine. It also
covers CVE requests and the follow-up work after an issue is filed.

The output is not "a bug report". The output is a small, self-contained
evidence package: exact version, exact build, exact input, exact stack, root
cause, impact, and a plausible fix.

## 1. Non-negotiable rules

1. **One root cause per issue.** Two different unchecked lengths, two
   different sinks, or two different parser stages are separate issues even if
   they live in the same function. If one fix would not fix both, split them.
2. **Reproduce on unmodified upstream code.** Check out the exact commit or
   release tag, verify `git status` is clean, and build from that tree. A
   finding that only reproduces with local patches is not ready.
3. **A stranger must be able to reproduce it.** The issue must contain enough
   source, build flags, a POC, and a run command that someone with a normal
   Linux toolchain can reproduce it. Do not depend on a private server, a
   private binary, a private corpus, or a non-public dataset.
4. **Evidence before prose.** Exact commit, compiler, sanitizer, POC SHA-256,
   exit code, and stack are more important than a long narrative.
5. **Never publish internal paths or secrets.** Replace `/root/...`,
   `/tmp/...`, hostnames, usernames, API tokens, and private URLs with
   repository-relative paths or neutral placeholders. The public issue must
   contain zero server paths.
6. **Do not overclaim.** Say "out-of-bounds read", not "RCE", unless you
   demonstrated control of the write or the instruction pointer. Say
   "sanitizer-only UB" when a normal build does not crash. Say "DoS" when the
   only demonstrated effect is a crash.
7. **Classify precisely.** "Buffer overflow" is not a synonym for every
   sanitizer finding. Use the actual class: stack-buffer-overflow read/write,
   heap-buffer-overflow read/write, use-after-free, integer overflow,
   null-pointer dereference, uninitialized read, uncontrolled allocation,
   uncontrolled recursion, shift UB, etc.
8. **Check duplicates first.** Search the project's issues and PRs, the CVE
   list, OSS-Fuzz, and ARVO. A known benchmark or already-fixed bug should not
   be filed as a new vulnerability.
9. **Follow the project's template.** If the repository has an issue form,
   fill its fields exactly. Do not paste a generic report into a form with
   required fields.
10. **Use the intended GitHub account.** Run `gh auth status` before posting.
    Do not post from an unrelated logged-in account.

## 2. End-to-end workflow

### Step 0 - Triage the finding

Before writing anything, decide what layer failed:

- **target/library bug**: the bad value is produced or consumed inside the
  project code; this is reportable.
- **harness bug**: the driver passes an invalid pointer, wrong length, or
  violates the public API contract; fix the harness, do not file.
- **environment bug**: ASan/UBSan instrumentation artifact, missing runtime,
  stack-limit issue, or a crash in libc/allocator outside the target; usually
  not a project vulnerability.
- **known/benchmark bug**: ARVO, OSS-Fuzz, or an existing CVE/issue; do not
  file as new.

Record the evidence for that classification. If you cannot tell, do not file
yet.

### Step 1 - Pin the version

```bash
git clone <repo> project
cd project
git checkout <full-commit-sha>
git status --short          # must be empty
git log -1 --format='%H %ci %s'
```

Record the exact commit and, if available, the release version from the
project's version header. Do not use a stale version string from a generated
file if the checked-out source does not contain that version.

### Step 2 - Reproduce from clean source

Build a **minimal reproducer** that reaches the same public API as the real
application. Prefer:

- the project's own fuzz target or a small public-API driver;
- only the translation units needed for the affected path;
- `-g -O1 -fno-omit-frame-pointer`;
- `-fsanitize=address,undefined -fno-sanitize-recover=all` when possible;
- a separate non-sanitized build to check whether the bug is a real crash or
  only a sanitizer observation.

If linking unrelated project symbols is a problem, use section garbage
collection (`-ffunction-sections -fdata-sections -Wl,--gc-sections`) or a
documented linker escape hatch such as
`-Wl,--unresolved-symbols=ignore-all`. State that this is only to drop unused
paths, not to hide a link error on the vulnerable path.

### Step 3 - Minimize and stabilize the POC

1. Shrink the input while preserving the same sanitizer stack and summary.
2. Run it at least three times. It must be deterministic.
3. Run a benign control input through the same binary. It must exit cleanly.
4. Record:
   - exact size in bytes;
   - SHA-256;
   - hex dump or `printf ... | xxd -r -p` recipe;
   - a byte-layout table (offset, meaning);
   - exit code and signal;
   - full sanitizer stack (not just the first frame).
5. If the crash depends on stack size, sanitizer, optimization, or
   instrumentation, say so explicitly and give the exact condition.

### Step 4 - Find the root cause

Trace the attacker-controlled value from input to sink:

```text
input field -> parser -> validation (or missing validation) -> sink
```

Record the exact source lines and the call chain. Explain:

- what the code assumes;
- why the assumption is false for this input;
- what the safe bound or invariant should be;
- whether the same missing check affects other callers.

A useful root-cause statement is one sentence plus a call chain. Avoid vague
phrases such as "the parser is unsafe".

### Step 5 - Search for duplicates and known status

Search:

- project issues and pull requests for the function name and sink;
- the CVE list (`https://www.cve.org`);
- OSS-Fuzz issue trackers where public;
- ARVO or benchmark identifiers if the task came from one.

If it is already fixed, identify the fixing commit. If it is a known
benchmark/ARVO sample, do not file it as a new vulnerability. If it is a
variant of a known bug, say so and explain what is different.

### Step 6 - Decide severity and CWE

Use the demonstrated effect, not the scariest theoretical effect:

- **Critical/High**: reliable memory corruption with a plausible control path,
  or a write primitive into sensitive state.
- **Medium**: deterministic crash/DoS, stack exhaustion in realistic
  deployments, or an OOB read with limited disclosure.
- **Low**: sanitizer-only UB, a read that cannot be shown to affect execution,
  or a crash requiring unusual build flags/configuration.

Use the most specific CWE. Common mappings:

| Finding | CWE |
|---|---|
| stack read/write past an array | CWE-125 / CWE-121 |
| heap read/write past an allocation | CWE-125 / CWE-122 |
| use-after-free | CWE-416 |
| null dereference | CWE-476 |
| unchecked allocation size | CWE-789 / CWE-190 |
| signed overflow | CWE-190 |
| shift width UB | CWE-190 |
| uncontrolled recursion | CWE-674 |
| uncaught exception/abort | CWE-248 |
| reachable assertion | CWE-617 |

If two classifications are honest (for example, CWE-125 and CWE-121 because a
read precedes a possible one-byte store), list both and explain which one the
reported sanitizer actually caught.

## 3. The issue body

Use the project's form when one exists. Otherwise use this structure. It is
ordered so a maintainer can stop reading at any point and still have enough
to reproduce.

```markdown
# [Project] <bug class> in `<function>`: <trigger> (<CWE>)

## Hi,

One short paragraph: what you were fuzzing, what the bug is, what the
sanitizer/abort shows.

## Summary

What the code does, the unchecked value or missing invariant, and the
demonstrated result. Include:

- **Affected versions**: exact release/tag and commit
- **Severity**: one sentence with the demonstrated effect
- **CWE**: primary and secondary

## Detail

### Affected code

Paste the relevant function(s) with file:line references. Keep it short
enough to read; trim unrelated branches.

### Root cause

A precise explanation: input field -> parser -> missing check -> sink.
Explain why the safe invariant is not established for this input.

### Trigger path

A text call chain with file:line at each step.

## Build the reproducer

Exact clone/checkout, driver source, build command, dependencies, and the
tested OS/compiler. No private paths.

## POC (tested on the unmodified source)

Size, SHA-256, hex/printf recipe, decoded layout table, and run command.

## Trigger result

Exit code, signal, full stack, and the relevant ASan object/offset report.
State whether it is deterministic and whether a non-sanitized build also
crashes.

## Impact

Who can trigger it, what the demonstrated effect is, and what is *not*
demonstrated. Avoid "could be RCE" without a write/control argument.

## Suggested fix

A minimal safe check, with code. Mention the invariant and a regression test.
```

### Title formula

Use the form:

```text
[Project] <bug class> in <function>: <root cause / trigger> (CWE-xxx)
```

Examples:

- `[NetHack] Stack-buffer-overflow WRITE in get_plname_from_file: an unchecked save-file plname-size overflows char plbuf[PL_NSIZ_PLUS] (CWE-121)`
- `[Duktape] Native stack overflow in duk__match_regexp on a zero-width quantified backreference (CWE-674)`
- `[htslib] Signed integer overflow in bcf_get_version() on the VCF header path (CWE-190)`

Put the function and the trigger in the title. "Fix crash" is not a title.

### What to put in the first paragraph

A maintainer should learn, in one paragraph:

1. the target (project + component);
2. the input type (file, script, packet, API argument);
3. the bug class;
4. the sanitizer/abort result;
5. whether a normal build is affected.

Do not start with "I am a security researcher" or a long background essay.

## 4. Reproducer quality bar

A good reproducer has these properties:

- **Minimal**: only the code needed to reach the sink. If the project has a
  public API, use it. If not, use the smallest internal entry point with a
  clear explanation.
- **Pinned**: built against the exact commit/tag in the issue.
- **Self-contained**: no ROM, database, private corpus, or server process
  unless the issue explains how to obtain it publicly.
- **Deterministic**: same sanitizer summary and exit code on repeated runs.
- **Controlled**: a benign input exits cleanly through the same driver.
- **Honest**: if the crash needs a small stack (`ulimit -s`), say so; if it
  needs a sanitizer, say so; if it needs an unusual allocator, say so.
- **Portable enough**: prefer Linux x86_64 with clang; mention the exact
  compiler and OS you tested. If a linker workaround is required, explain it.

Do not ship a 200-line harness when a 20-line public-API driver suffices.
Do not hide build failures with `|| true`; the build must succeed and the
vulnerable path must be linked.

## 5. Handling difficult findings

### Sanitizer-only UB

If the non-sanitized build does not crash, do not call it a remote DoS.
Describe it as language-level UB and say which builds trap it. The issue is
still useful if the UB can miscompile or bypass a check; explain why.

### Stack overflow / recursion

A repeated stack trace is expected: every recursive frame is a separate
call at the same source line. Explain that the repetition is the evidence of
uncontrolled recursion. If the crash depends on stack size, give a table:

| build | stack limit | result |
|---|---|---|
| ASan | 2 MiB | stack-overflow, exit 134 |
| plain | 1 MiB | SIGSEGV, exit 139 |
| plain | 8 MiB | catchable RangeError, exit 0 |

### Uncaught exception / abort

For C++ `std::length_error`, `std::bad_alloc`, or similar, show both the
`terminate called after throwing ...` message and, if useful, a gdb
`catch throw` / `bt` trace. State whether a normal build aborts (usually yes).

### Format/version uncertainty

Do not claim a version is affected just because a generated header says so.
Verify the code at the tag/commit. If the bug is only in a development tree,
say "master at commit ...", not "all versions".

## 6. Project-specific forms

### FCEUX issue form

Fields: Title, GUI version, Describe the bug, Steps to reproduce, Expected
behavior, Environment, Screenshots, Additional context.

- GUI version: name the GUI (`Qt`, `SDL`, `Windows`) and the version/commit.
- Put the driver/build/POC in Additional context.
- Screenshots: terminal output of the sanitizer/abort.

### Duktape / C projects without a form

Use the standard body above. Keep the driver and build command in the issue;
Duktape reviewers expect a concrete `duk_eval_raw` or public-API path.

### GitHub Security Advisories

If the project accepts reports through **Security > Report a vulnerability**,
use that private channel first. A public issue can expose users before a fix.
Once an advisory/CVE is coordinated, publish only the agreed details.

## 7. CVE requests

### Where to submit

The MITRE CNA-of-Last-Resort request form has moved. The current entry points
are:

- CVE ID requests:
  `https://mitre.github.io/mitre-cve-roles/cve-id-request/`
- General CVE reporting entry:
  `https://www.cve.org/ReportRequest/ReportRequestForNonCNAs`
- Legacy form (still available for now, but being retired):
  `https://cveform-legacy.mitre.org/`

If the project has its own CNA (for example through GitHub Security
Advisories), use that CNA first. MITRE is the fallback when no CNA covers the
product.

### CAN vs CVE

The first request returns a `CAN-YYYY-#######` tracking number. A CAN is **not
a CVE ID** and must not be used publicly. The CVE ID is assigned later and
appears on the CAN record as:

```text
YOUR REQUEST WAS ACCEPTED AND ASSIGNED THIS CVE ID
```

### Submission flow

1. Open the CVE ID request form.
2. Log in with Google or GitLab.
3. Click **Reserve One CAN**.
4. Open the new CAN link in the ID column.
5. Fill the required fields:
   - Problem types (CWE)
   - Vendor or project
   - Product name
   - Version
   - Vulnerability Description
   - References URL
6. Add CVSS 4.0 or 3.1 if you have a defensible score. If not, remove the
   Metrics section with the red `x`.
7. Submit the request.
8. Wait for the CVE ID; do not publish the CAN number.
9. If the request is rejected or needs changes, reply with the missing
   evidence; do not open a duplicate CAN.

### CVE description content

A CVE description is not the full issue body. It should be self-contained and
short enough for the CVE record:

```text
<Project> <version/commit> has a <bug class> in <function/file>.
<One sentence describing the unchecked input or missing invariant and the
demonstrated memory-safety/availability effect.>
An attacker can trigger this by <input type>, for example <short POC
description>. This may lead to <demonstrated impact>.
```

Include:

- exact affected version/commit, or a range only if you verified it;
- CWE;
- the bug class and sink;
- the trigger input;
- demonstrated impact, not speculation;
- references to the public issue/PR/fix;
- credit name only if the reporter wants it.

Do not put internal hostnames, paths, or unverified exploit claims in a CVE
record.

### CVE request template

```text
Product: <project>
Version: <release / commit>
Vendor: <project maintainer or organization>
Problem type: <CWE-xxx: name>
Description: <self-contained description as above>
References: <issue URL>, <fix PR/commit URL>
Impact: <DoS / OOB read / OOB write / UAF / etc.>
Attack vector: <local file / network / API / script>
Credit: <name or anonymous>
```

### Version-attribution trap

A generated `*_version.h` or build metadata can be stale. Before claiming a
release is affected, check that the release tag actually contains the
vulnerable code. If only master is affected, say master. If the fix is already
in a release, do not list that release as vulnerable.

## 8. Posting and follow-up

### Before posting

- Run `gh auth status` and confirm the intended account.
- Check the repository's issue template and labels.
- Search for an existing issue/PR one more time.
- Verify every link in the body resolves.
- Verify the POC hash and size one more time.
- Remove internal paths and secrets.
- If the project has a private security channel, use it instead of a public
  issue.

### While the issue is open

- Answer technical questions with data, not opinions.
- If a maintainer asks for a different build, reproduce it and report the
  result.
- If CI fails, check whether the failing job is related to your change. A
  failure in an unrelated test is not a reason to modify unrelated code.
- For flaky CI, provide the failing job, the exact assertion, and evidence
  that the same test passes on `main`; ask for a rerun.
- If a maintainer finds the root cause is different, update the issue rather
  than arguing about the original narrative.

### After a fix

- Verify the fix against the original POC.
- Add a regression test if the project wants one.
- Request a CVE only after the maintainers agree on the fix/affected versions,
  unless the project has no security process and the issue is already public.
- Update the CVE request with the fix commit and public references.

## 9. Publication hygiene

Before publishing, scan the document for:

- `/root`, `/tmp`, `/home`, `C:\Users`, hostnames, IP addresses;
- private repository names, internal task IDs, workspace paths;
- tokens, cookies, API keys, email addresses;
- personal names not intended as credit;
- screenshots that reveal any of the above;
- stack traces with absolute build paths (rewrite to repository-relative).

Use placeholders in logs:

```text
# before
#0 foo /root/task/workspace/project/src/foo.c:42

# after
#0 foo src/foo.c:42
```

Keep the original evidence locally. Publish only the minimized, sanitized
version.

## 10. Checklists

### Pre-flight

- [ ] Finding is in target code, not harness/environment.
- [ ] Reproduced on unmodified upstream commit/tag.
- [ ] `git status` clean at that commit.
- [ ] Root cause traced to a specific sink with file:line.
- [ ] POC minimized and deterministic (3/3).
- [ ] Benign control passes.
- [ ] Non-sanitized behavior checked.
- [ ] Duplicate/CVE/ARVO search done.
- [ ] Severity and CWE chosen from demonstrated effect.

### Pre-post

- [ ] Title follows the project's convention and names the function.
- [ ] Affected version/commit is exact.
- [ ] Build command is complete and was tested from a clean tree.
- [ ] POC size, SHA-256, and layout table are present.
- [ ] Full stack and exit code are present.
- [ ] Impact distinguishes demonstrated vs theoretical.
- [ ] Suggested fix names the invariant and includes a regression test.
- [ ] No internal paths, secrets, or private data.
- [ ] Correct GitHub account is active.
- [ ] Project security channel checked.

### Pre-CVE

- [ ] Maintainers agree on affected versions and root cause.
- [ ] Public issue/PR/fix references exist or are coordinated.
- [ ] CWE and CVSS are defensible.
- [ ] Description is self-contained and free of internal details.
- [ ] CAN is kept private until the CVE ID is assigned.
- [ ] Backport branches are listed only if verified.

## 11. Reusable templates

See:

- `templates/github-issue.md` — generic issue body.
- `templates/cve-request.md` — CVE request fields.
- `checklists/pre-flight.md` — investigation checklist.
- `checklists/pre-submit.md` — publication checklist.

## 12. Lessons from real reports

These are the recurring failure modes worth remembering.

### One root cause, one issue

A project can have many crashes from one missing check. Conversely, one
function can contain several independent missing checks. Group by "would one
fix close all of these?" If not, split. This is why a save-file parser can
produce both a length-driven WRITE and a terminator-driven READ in the same
function: they need different fixes.

### Do not call every finding a buffer overflow

Classify the sanitizer's actual operation:

- `READ of size N` -> out-of-bounds read;
- `WRITE of size N` -> out-of-bounds write;
- use-after-free -> UAF;
- signed overflow -> integer overflow/UB;
- null member access -> null dereference;
- recursion -> uncontrolled recursion/stack exhaustion.

The maintainer's first question is often "is this a real memory-safety bug or
just UBSan?" Answer it in the issue.

### Reproducibility is a feature of the report

A stack trace from a private fuzzer binary is not reproducibility. Include a
driver, exact build, and input. If the crash needs a special stack size or
sanitizer, document it. If the harness is the only caller, say so and show
the public-API path.

### Be conservative about exploitability

Demonstrated crash != RCE. Demonstrated OOB read != information disclosure
unless you show the data leaving the process. Demonstrated OOB write !=
control-flow hijack unless you show control. Maintainers trust precise
reports more than dramatic ones.

### CI failures can be unrelated

A failing macOS job in an unrelated TLS test is not evidence that the SDP fix
is wrong. Identify the exact assertion and check whether it passes on `main`.
Do not "fix" unrelated tests inside a security PR.

### Version claims need verification

A stale generated version header can make you claim the wrong release is
affected. Check the tag, not a build artifact. If a fix is already in a
release, remove that release from the affected list.

### Keep the evidence package small

A maintainer should be able to reproduce in minutes:

1. clone;
2. checkout;
3. paste driver;
4. compile;
5. create POC from a hex string;
6. run one command.

If your instructions need a private server or ten manual steps, simplify
before posting.
