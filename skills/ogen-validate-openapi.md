---
name: Validate an OpenAPI v3 document against the ogen parser
description: >-
  Check whether an OpenAPI v3 document is accepted by ogen — in the browser with
  the WASM validator, or locally with the CLI — and read the location-anchored
  errors it returns.
api: cli/ogen-cli.yml
sandbox: sandbox/ogen-sandbox.yml
grounding: >-
  Verified against ogen.dev/docs/validator, ogen.dev/docs/config and
  cmd/ogen/main.go. ogen exposes no HTTP API; this skill is grounded in the CLI
  and the published WASM validator.
operations:
  - ogen --strict
  - ogen --target
  - ogen --initialisms
generated: '2026-08-06'
method: generated
---

# Validate an OpenAPI v3 document with ogen

## Fastest path: the browser validator

https://ogen.dev/docs/validator

No signup, no key, no upload. It runs the real ogen parser and IR builder
compiled to WebAssembly — the same pipeline the generator uses — entirely
client-side. Errors come back with source locations in exactly the form the CLI
prints them.

Scope limit to state plainly when reporting a result: the validator focuses on
schema validity and ignores operations ogen cannot yet generate, equivalent to
`generator.ignore_not_implemented: ["all"]`. **A valid result does not mean
every operation will generate.** Confirm with a real generation run before
claiming support.

## Local path: generate into a throwaway target

There is no dedicated `validate` subcommand. Generation *is* the check:

```sh
go tool ogen --target /tmp/ogen-check --clean spec.yml
```

A clean exit means ogen parsed the document and produced code for every
operation. Any failure prints the offending field with its file and location.

Tighten or loosen the check with flags:

- `--strict` disables cross-type constraint interpretation — `pattern` on a
  number or `min`/`max` on a string become rejections rather than being
  reinterpreted. Use it to find sloppy constraints.
- `--initialisms ID,URL,API` replaces the initialism set (add `inherit` to keep
  the built-in one, or pass empty to disable all); `--initialisms-extra FQDN`
  adds on top. Both are repeatable or comma-separated. Use these when generated
  identifier casing is the thing under review.
- `--version` prints the generator version — always report it alongside a
  verdict, because acceptance is version-dependent.

## Separating "invalid spec" from "not yet implemented"

These are different findings and must not be reported as one:

1. Run without `ignore_not_implemented`. Failures here are a mix of both.
2. Re-run with `generator.ignore_not_implemented: ["all"]` in a config file
   passed via `--config`. Anything that still fails is a genuine spec problem;
   anything that now passes was an unsupported construct, not invalid OpenAPI.

Naming the specific unsupported case instead of `"all"` keeps the rest of the
document strict.

## Version context

ogen targets OpenAPI v3. The 3.1 object model is what the source is documented
against, and 3.1 nullable-via-type-array support has landed, but 3.1 coverage is
partial — see `conformance/ogen-conformance.yml`. If a 3.1-only construct
fails, check the issue tracker before calling the document invalid.

## Sources

- https://ogen.dev/docs/validator
- https://ogen.dev/docs/config
- https://github.com/ogen-go/ogen/blob/main/cmd/ogen/main.go
