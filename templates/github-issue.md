# [Project] <bug class> in `<function>`: <trigger> (CWE-xxx)

## Hi,

<What you were fuzzing/testing, the input type, the bug class, and the
sanitizer/abort result. One paragraph.>

## Summary

<What the code does and the missing check/invariant. Then:>

- **Affected versions**: <release/tag and/or commit>
- **Severity**: <demonstrated effect, one sentence>
- **CWE**: <primary> (<name>), <secondary if honest>

## Detail

### Affected code

```c
/* file:line */
<relevant function, trimmed>
```

### Root cause

<input field -> parser -> missing check -> sink. Explain why the invariant
does not hold for this input.>

### Trigger path

```text
entry (file:line)
  -> function (file:line)
     -> sink (file:line)
```

## Build the reproducer

```bash
git clone <repo> project
cd project
git checkout <full-commit-sha>
```

Save as `repro.c`:

```c
<minimal public-API driver>
```

```bash
<exact compiler command>
```

<OS, compiler, sanitizer, dependencies. No private paths.>

## POC (tested on the unmodified source)

<size> bytes:

```bash
printf '%s' '<hex>' | xxd -r -p > poc.bin
sha256sum poc.bin
# <sha256>
ls -l poc.bin
```

| byte range | meaning |
|---|---|
| 0-3 | <field> |
| 4-... | <payload> |

Run:

```bash
ASAN_OPTIONS=detect_leaks=0:abort_on_error=1:symbolize=1 ./repro poc.bin
```

## Trigger result

<exit code, signal, deterministic or not, sanitizer stack, object/offset>

```text
<full stack>
```

<State whether a non-sanitized build also crashes.>

## Impact

<Who can trigger it, what is demonstrated, what is not demonstrated.>

## Suggested fix

```c
<minimal safe check>
```

<Invariant, regression test, and any related callers.>
