# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-11-09

Requires Koja 0.20.1.

### Added

- `Conn`, one HTTP exchange as a value. It starts as the parsed request and accumulates the response through `resp`, `text`, `json`, `put_resp_header`, and `halt`. `to_response` converts it into an `HTTP.Response` with the request's HTTP version.
- `Router`, an immutable route table built with `get`, `post`, `put`, `patch`, `delete`, and `route`. The first registered match wins. `use` appends middleware that runs before the handler, and a halted conn stops the pipeline.
- Path parameters bound from `:name` pattern segments and read with `conn.param`. Each segment is percent decoded on its own, so an encoded slash stays inside its segment.
- Query parameters decoded once into `conn.query_params`, a list of pairs in wire order. `conn.query_param` returns the first value and `conn.query_values` returns every value. `+` reads as a space, and a key with no `=` reads as `""`.
- `Rejection`, the reason the router answered a request itself. `BadRequest` for a path or query that does not decode, `MethodNotAllowed` with the methods that do have a route, and `NotFound`. `Router.on_reject` replaces the plain text default with one function for all three.
- 405 responses carry a sorted, deduplicated `Allow` header.
