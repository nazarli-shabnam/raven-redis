# raven-redis

A Redis client for the [Raven](https://martian56.github.io/raven/) programming language.

> ⚠️ **Pre-release / work in progress.** API may change until `v0.1.0` is tagged.

## Quick start

```raven
import "github.com/martian56/raven-redis" { connect }

fun main() {
    match connect("127.0.0.1:6379") {
        Ok(client) -> {
            client.set("greeting", "hello")
            match client.get("greeting") {
                Ok(Some(v)) -> print(v),       // "hello"
                Ok(None)    -> print("(nil)"),
                Err(e)      -> print(e.message),
            }
            client.close()
        }
        Err(e) -> print(e.message),
    }
}
```

## Running any command

The typed helpers cover common cases; for everything else, use the generic
binary-safe command runner:

```raven
// command(args) runs ANY Redis command and returns a Value.
let reply = client.command(["LPUSH", "mylist", "a", "b", "c"])?
let n = client.command(["LLEN", "mylist"])?
```

A Redis error reply (e.g. `-WRONGTYPE ...`) is returned as an `Err`, so `?` works naturally.

## Connecting with options

```raven
let opts = ConnectOptions.new("127.0.0.1", 6379)
    .password("s3cr3t")        // AUTH
    .username("appuser")       // ACL username (optional)
    .db(2)                     // SELECT 2
    .client_name("my-app")     // CLIENT SETNAME
let client = connect_opts(opts)?
```

## Pipelining

```raven
let p = client.pipeline()
p.queue(["SET", "a", "1"])
p.queue(["INCR", "a"])
p.queue(["GET", "a"])
let replies = p.exec()?   // List<Value>, one per queued command
```

## Reply values

`command()` returns a `Value`:

```raven
enum Value {
    Simple(String),       // +OK
    Error(String),        // -ERR ...  (top-level errors surface as Err instead)
    Int(Int),             // :123
    Bulk(String),         // $... bulk string (binary-safe)
    Null,                 // $-1 / *-1
    Array(List<Value>),   // *...
}
```

Accessor helpers: `as_string()`, `as_int()`, `as_array()`, `is_null()`.

## Typed helpers (v1)

`ping`, `echo`, `set`, `get`, `del`, `exists`, `expire`, `ttl`, `incr`, `decr`.
Everything else: use `command(args)`.

## Limitations (v1)

- **RESP2 only.** RESP3 is **not implemented yet** (planned for a future release).
- **No TLS.** Raven's `std/net` has no TLS support, so connections are plaintext only.
- **Single connection, no pooling.** Raven v2 has no concurrency.

## Testing

- Unit tests (no network) cover the RESP encoder and parser, including split-buffer / partial-read cases:

  ```
  rvpm test
  ```

- Integration tests run against a live server at `127.0.0.1:6379` (start one with Docker:
  `docker run --rm -p 6379:6379 redis:7`).

## License

MIT — see [LICENSE](LICENSE).
