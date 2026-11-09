# Agent Guide

This is a [Koja](https://kojalang.org) project. Koja is a statically
typed, compiled language with Ruby-inspired syntax, value semantics,
and Erlang-style concurrency.

## This package

`Trail` routes parsed HTTP requests to responses. It owns no sockets.
The public surface is `Conn`, `Router`, `Route`, and `Rejection`, one
per file in `src/`. `default_reject` is public only because the
`reject` field default names it, and `@doc false` keeps it out of the
docs.

`Conn` is a value. Handlers and middleware are `fn (Conn) -> Conn`,
and dependencies reach a handler by closure capture, never through a
slot on the conn. `Router.dispatch` decodes the path and query once,
before middleware, into `path_segments` and `query_params`. The raw
strings stay in `path` and `query`.

The router answers three requests itself, each as a `Rejection`. A
path or query that does not decode is a 400, a path with routes for
other methods only is a 405 with an `Allow` header, and no match is
a 404. One hook, `on_reject`, renders all three.

Tests mirror `src/`. `test/conn_test.koja` covers request reads and
response writes, `test/router_test.koja` covers matching, middleware,
and rejections, and `test/rejection_test.koja` covers the status map.
Shared helpers live in `test/support/`. `koja test` compiles every
file under `test/`, so a `priv` declaration there is visible to every
test file, and `priv` functions in `src/` are visible to tests.

`CHANGELOG.md` follows Keep a Changelog.

## Look up documentation

`koja doc <symbol>` prints the full doc for any stdlib or project
symbol, like `koja doc List.append`. `koja doc search <query>`
lists every symbol whose name or doc mentions the query.

The standard library sources are extracted to `~/.koja/stdlib/`,
one directory per compiler build. Read or grep them there when the
docs are not enough.

## Commands

Set `KOJA_DIAGNOSTICS=short` to print each diagnostic on one line.

- `koja check` type checks without compiling
- `koja test` runs `test "description"` blocks from `src/` and
  `test/`. `assert expr` fails the test when `expr` is false, and
  `fail "message"` fails it outright. Reaching `end` passes.
- `koja run` builds and executes the project
- `koja format` formats the project in place (`--check` to verify)

## Language essentials

- No `let`, `var`, `mut`, or semicolons. Assignment creates a
  variable, and blocks close with `end`.
- No `else if`. Use `cond` for multi-branch conditionals.
- Values never alias. A mutating function takes `self` and returns a
  new value, so rebind the result, as in `list = list.append(42)`.
- `-> T ! E` declares a fallible function. `try expr` unwraps or
  propagates the error, `fail e` returns an error, and
  `expr rescue e -> handler` handles it inline.
- Files in one package share a namespace. Same-package code needs no
  imports. Use `alias Pkg.Type` to shorten external package paths.
