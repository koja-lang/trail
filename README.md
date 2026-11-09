# Trail

[![CI](https://github.com/koja-lang/trail/actions/workflows/ci.yml/badge.svg)](https://github.com/koja-lang/trail/actions/workflows/ci.yml)
[![GitHub Release](https://img.shields.io/github/v/release/koja-lang/trail)](https://github.com/koja-lang/trail/releases)

Composable HTTP routing for [Koja](https://github.com/koja-lang/koja).
One request is one immutable `Conn` value, and every handler and
middleware is a plain function from `Conn` to `Conn`.

Trail routes parsed requests to responses and owns no sockets. A
server package supplies the `HTTP.Request` and writes the
`HTTP.Response`.

## Features

- Route table built with chainable verb functions, matched first registration wins
- Path parameters bound from `:name` segments, percent decoded, read with `conn.param`
- Query parameters decoded once into a list of pairs, read with `conn.query_param` and `conn.query_values`
- Middleware runs in order before the matched handler, and `conn.halt()` stops the pipeline
- The router answers 400, 404, and 405 itself, each as a `Rejection` rendered by one replaceable hook
- 405 responses carry a sorted `Allow` header, as RFC 9110 requires

## Installation

Add the package to your `koja.toml`.

```toml
[dependencies]
trail = { github = "koja-lang/trail", tag = "v0.1.0" }
```

## Usage

```koja
alias HTTP.Request
alias HTTP.Response
alias HTTP.Status
alias Trail.Conn
alias Trail.Rejection
alias Trail.Router

fn health(conn: Conn) -> Conn
  conn.text(Status.ok(), "healthy")
end

fn show_session(conn: Conn) -> Conn
  id = conn.param("id").or("")
  conn.text(Status.ok(), "session " <> id)
end

fn require_auth(conn: Conn) -> Conn
  match conn.get_header("Authorization")
    Option.Some(_) -> conn
    Option.None -> conn.text(Status.unauthorized(), "no credentials").halt()
  end
end

fn reject(conn: Conn, rejection: Rejection) -> Conn
  conn.text(rejection.status(), "rejected: #{rejection.status().reason}")
end

fn build_router -> Router
  Router{}
    .use(&require_auth/1)
    .on_reject(&reject/2)
    .get("/health", &health/1)
    .get("/sessions/:id", &show_session/1)
end

fn serve(router: Router, request: Request) -> Response
  router.dispatch(request)
end
```

## Design

- **`Conn` is a value.** Handlers transform it and return it. There is no hidden connection state, so handlers are plain functions that tests can call directly.
- **Dependencies arrive by closure capture.** `Conn` carries no application context slot. A handler that needs a store captures it when the route table is built, so each handler declares exactly what it uses.
- **Middleware and handlers share one shape**, `fn (Conn) -> Conn`. Middleware registered with `use` runs in order before the matched handler. `conn.halt()` stops the pipeline and sends the response as-is.
- **Decoding happens once, before middleware.** `Router.dispatch` fills `conn.path_segments` and `conn.query_params` with the decoded forms and keeps `conn.path` and `conn.query` as the wire strings. A request that does not decode never reaches middleware.

## Responses

`Conn` builds responses incrementally.

| Function                       | Effect                                    |
| ------------------------------ | ----------------------------------------- |
| `resp(status, body)`           | Set status and body, headers untouched.   |
| `text(status, body)`           | Plain text body with content headers.     |
| `json(status, value)`          | JSON body with content headers.           |
| `put_resp_header(name, value)` | Set one response header.                  |
| `halt()`                       | Stop the pipeline after the current step. |

`to_response` converts the accumulated state into an `HTTP.Response` with the request's HTTP version. A conn that never set a status becomes a 500, because a handler forgot to respond.

## Rejections

The router answers three kinds of request without running a handler. Each is a `Trail.Rejection`.

| Variant                          | Status | When                                                          |
| -------------------------------- | ------ | ------------------------------------------------------------- |
| `BadRequest(URI.Error)`          | 400    | The path or query has malformed percent encoding or bad UTF-8 |
| `MethodNotAllowed(List<Method>)` | 405    | Routes exist for the path, but none for the method            |
| `NotFound`                       | 404    | No route matches the path                                     |

The default rendering is the status reason as plain text. `on_reject` replaces it with one function for all three, so an API can render every error in its own JSON shape. For `MethodNotAllowed` the router sets the `Allow` header before calling the hook, and the variant carries the same methods for use in a body.

## Query parameters

`?tag=a&tag=b&q=x+y` decodes to `[("tag", "a"), ("tag", "b"), ("q", "x y")]` in wire order. Nothing is dropped.

- `conn.query_param("tag")` is `Some("a")`, the first value.
- `conn.query_values("tag")` is `["a", "b"]`.
- `+` reads as a space, as in form encoding.
- `?flag` and `?flag=` both read as `""`.
- Keys are literal. `tag[]=a` is the parameter named `tag[]`.

## Not yet supported

- Route groups and mounted sub-routers.
- `HEAD` falling through to `GET` routes.
- Request body streaming. `Conn.body` is the fully buffered body.
- The `tag[]` and `user[name]` query conventions. The flat pair list is the substrate for a later form decoder.

## Development

```sh
koja test
```

## License

Copyright (c) 2026 Henry Popp

This project is MIT licensed. See the [LICENSE](LICENSE) for details.
