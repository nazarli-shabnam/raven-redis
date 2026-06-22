# raven-redis

A Redis client for the [Raven](https://martian56.github.io/raven/) programming language.

> Current release: **v0.3.0**. Pre-1.0, so the API may still evolve between minor versions.

## Installation

Add the dependency to your `rv.toml`:

```toml
[dependencies]
"github.com/nazarli-shabnam/raven-redis" = "0.3.0"
```

or from the command line:

```
rvpm add github.com/nazarli-shabnam/raven-redis@v0.3.0
```

## Quick start

```raven
import "github.com/nazarli-shabnam/raven-redis" { connect }

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

## RESP3

RESP3 (Redis 6+) is the newer protocol with richer, self-describing reply
types (maps, sets, doubles, booleans, …). It's **opt-in**: a connection speaks
RESP2 unless you upgrade it. Either set the option at connect time —

```raven
let opts = ConnectOptions.new("127.0.0.1", 6379).resp3(true)
let client = connect_opts(opts)?   // sends HELLO 3 (folds in AUTH / SETNAME)
```

— or upgrade an existing plain connection:

```raven
let info = client.hello3()?        // returns the server's handshake Map
```

Once in RESP3 mode the server uses the extra reply types: e.g. `HGETALL`
returns a `Map`, `SMEMBERS` a `Set`, doubles arrive as `Double`. Requires
Redis 6+ (`hello3` / `resp3` error on older servers).

## Pipelining

Send several commands in one round trip and get one reply per command:

```raven
let replies = client.pipeline([
    ["SET", "a", "1"],
    ["INCR", "a"],
    ["GET", "a"],
])?
// replies : List<Value>, one entry per command, in order.
```

Pipeline replies are returned **raw** (a Redis error appears as `Value.Error`
rather than aborting), so one failing command never desyncs the rest.

## Reply values

`command()` returns a `Value`. To destructure it, import the type from the
`resp` submodule:

```raven
import "github.com/nazarli-shabnam/raven-redis/resp" { Value }

enum Value {
    // RESP2
    Simple(String),            // +OK
    Error(String),             // -ERR ...  (top-level errors surface as Err instead)
    Int(Int),                  // :123
    Bulk(String),              // $... bulk string (binary-safe)
    Null,                      // $-1 / *-1  (or _ in RESP3)
    Array(List<Value>),        // *...
    // RESP3
    Bool(Bool),                // #t / #f
    Double(Float),             // ,3.14
    BigNumber(String),         // (1234...   (kept as text)
    Verbatim(String, String),  // =...  -> (format, text)
    Map(List<Value>),          // %...  -> flat key,value,key,value,...
    Set(List<Value>),          // ~...
    Push(List<Value>),         // >...  (out-of-band / pub-sub)
}
```

Accessor helpers: `as_string()` (Simple/Bulk/Verbatim), `as_int()`, `as_bool()`,
`as_double()`, `as_array()` (Array/Set/Push), `as_map()`, `is_null()`.

## Typed helpers

Strings / counters: `set`, `set_ex`, `set_nx`, `get`, `mset`, `mget`,
`incr`, `decr`, `incr_by`, `decr_by`.

Keys / lifetime: `del`, `exists`, `expire`, `persist`, `ttl`, `type_of`,
`keys`, `scan`.

Connection: `ping`, `echo`, `close`.

Everything else: use `command(args)`. `scan` returns a `ScanResult { cursor, keys }`
— iterate from cursor `0` until it returns `0` again:

```raven
let cursor = 0
let done = false
while done == false {
    let step = client.scan(cursor, "user:*", 100)?
    for k in step.keys {
        print(k)
    }
    cursor = step.cursor
    if cursor == 0 {
        done = true
    }
}
```

## Limitations

- **No TLS.** Raven's `std/net` has no TLS support, so connections are plaintext only.
- **Single connection, no pooling.** Raven v2 has no concurrency.
- **No pub/sub API yet.** RESP3 `Push` frames are parsed, but there's no
  subscription loop helper.
- **RESP3 attributes** (`|`) are parsed and skipped, not surfaced.
- **inf / -inf / nan doubles** are not supported (rare in practice).

## Testing

- Unit tests (no network) cover the RESP encoder and parser, including split-buffer / partial-read cases:

  ```
  rvpm test
  ```

- Integration tests run against a live server at `127.0.0.1:6379` (start one with Docker:
  `docker run --rm -p 6379:6379 redis:7`).

## License

MIT — see [LICENSE](LICENSE).
