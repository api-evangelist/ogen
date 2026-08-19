---
name: Generate a Go client and server from an OpenAPI v3 spec with ogen
description: >-
  Take an OpenAPI v3 document and produce a statically typed Go client, server,
  router and validators using the ogen code generator, pinned reproducibly in
  go.mod.
api: cli/ogen-cli.yml
contract: json-schema/ogen-config.jsonschema.json
grounding: >-
  Every flag, config key and feature name below is verified against
  cmd/ogen/main.go, gen/features.go and schemas/ogen.jsonschema.json in
  github.com/ogen-go/ogen. ogen exposes no HTTP API, so this skill is grounded
  in the CLI surface and the published configuration JSON Schema instead of an
  operationId set.
operations:
  - ogen --target
  - ogen --package
  - ogen --clean
  - ogen --config
generated: '2026-08-06'
method: generated
---

# Generate a Go client and server with ogen

ogen is a build-time generator, not a service. Nothing is called over the
network; you run one command against one OpenAPI v3 document and it writes Go
source.

## 1. Prepare the module

```sh
go mod init <project>
```

## 2. Pin the generator

Go 1.24 or newer. Use the tool directive so the generator version is recorded in
`go.mod` and the output is reproducible across the team and in CI:

```sh
go get -tool github.com/ogen-go/ogen/cmd/ogen@latest
```

`go install -v github.com/ogen-go/ogen/cmd/ogen@latest` also works but pins
nothing. A container invocation is available when Go is not on the box:

```sh
docker run --rm --volume ".:/workspace" ghcr.io/ogen-go/ogen:latest \
  --target workspace/petstore --clean workspace/petstore.yml
```

## 3. Declare generation

Create `generate.go`:

```go
package project

//go:generate go tool ogen --target petstore --clean petstore.yml
```

Then `go generate ./...`.

- `--target` (default `api`) is the output directory.
- `--package` (default `api`) is the Go package name.
- `--clean` removes previously generated files first — always set it, otherwise
  renamed operations leave orphaned `*_gen.go` files behind.
- The spec path is a positional argument, not a flag: `ogen [options] <spec>`.

## 4. Add a config file when the spec is non-trivial

Pass it with `--config`. Add the schema annotation on line 1 so editors with the
YAML language server give you completion and validation:

```yaml
# yaml-language-server: $schema=https://raw.githubusercontent.com/ogen-go/ogen/main/schemas/ogen.jsonschema.json
parser:
  infer_types: true
generator:
  features:
    enable:
      - paths/client
      - paths/server
      - ogen/otel
    disable_all: true
  ignore_not_implemented: ["all"]
```

Only two top-level sections exist: `parser` (how the document is read) and
`generator` (what code is produced). `additionalProperties` is false in the
schema — an unknown key is an error, not a warning.

- `parser.infer_types: true` detects a schema's type from its properties.
  Required for large real-world specs such as GitHub and Kubernetes.
- `parser.allow_remote` enables remote `$ref` resolution. Off by default.
- `parser.depth_limit` defaults to 1000.
- `generator.ignore_not_implemented: ["all"]` skips operations ogen cannot yet
  generate instead of failing on the first one. Prefer naming the specific case
  (e.g. `["nested objects in form parameters"]`) so the rest of the spec stays
  strict.
- `generator.filters.path_regex` and `generator.filters.methods` generate a
  subset of the spec.

## 5. Choose the feature set

Features resolve in three steps: start from the defaults (unless
`disable_all: true`), apply `disable`, then apply `enable`. `enable` always
wins.

Default set: `paths/client`, `paths/server`, `webhooks/client`,
`webhooks/server`, `ogen/otel`, `ogen/unimplemented`.

Also available: `client/security/reentrant`, `client/request/options`,
`client/request/validation`, `client/editors`, `server/response/validation`.

Client only:

```yaml
generator:
  features:
    enable:
      - client/editors
      - client/request/options
      - ogen/otel
      - ogen/unimplemented
      - paths/client
      - webhooks/client
    disable_all: true
```

Drop OpenTelemetry and its dependency with `disable: ["ogen/otel"]`.

## 6. Implement the server

ogen writes a `Handler` interface derived from the spec's operations. Implement
it and hand it to the generated constructor:

```go
srv, err := petstore.NewServer(service)
if err != nil {
    log.Fatal(err)
}
http.ListenAndServe(":8080", srv)
```

The generated `Server` implements `http.Handler`, so any `net/http` middleware
composes with it.

## Rules an agent should follow

- Never hand-edit `oas_*_gen.go`. Change the spec or the config and regenerate.
- Prefer spec `x-ogen-*` extensions over post-generation edits — see
  `vocabulary/ogen-extensions.yml` for the full 12-term registry.
- Optional and nullable fields are `OptT` / `NilT` / `OptNilT` wrappers, not
  pointers. Read with `v, ok := field.Get()`; write with `NewOptT(...)` or
  `SetToNull()`.
- `duration` maps to Go `time.Duration`, which is a documented divergence from
  the RFC 3339 duration JSON Schema defines. Do not assume RFC 3339 durations
  round-trip.
- Regeneration is not a release. Bumping the pinned ogen version can change
  generated identifiers; run it deliberately and review the diff.

## Sources

- https://ogen.dev/docs/intro
- https://ogen.dev/docs/config
- https://ogen.dev/docs/features
- https://github.com/ogen-go/ogen/blob/main/cmd/ogen/main.go
- https://github.com/ogen-go/ogen/blob/main/gen/features.go
