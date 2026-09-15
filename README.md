# Http Implementation with code quality

A minimal HTTP server written in C. It listens on TCP port `12345`, serves files from
`client/files/` and answers a subset of the HTTP methods using a pre-allocated thread pool.

## What the program does

1. `main()` spawns **20 threads** (`THREAD_POOL_SIZE`), then opens a server socket
   (`AF_INET`/`SOCK_STREAM`) bound to `INADDR_ANY:12345`.
2. Every accepted connection is pushed onto a **linked-list queue**
   (`utils/myqueue.c`, `enqueue`/`dequeue`) under a `pthread_mutex`, then a
   `pthread_cond_signal` wakes up one thread from the pool.
3. An idle thread pops a client socket and calls `handle_connection()`:
   read the request (4096-byte buffer, up to the first `\n`), parse, respond, `close()`.
4. `fill_request()` (`utils/http.c`) splits the request line with `strtok_r` and fills
   the global `httprq` struct with the **method** and the **URL**.
5. Routing is a plain string comparison on the method, then `sendResponse()` writes the
   header followed by the file contents to the socket.

### Method routing

| Method   | File served (from `client/files/`)   | Status returned |
|----------|--------------------------------------|-----------------|
| `GET`    | the file named in the URL            | `200 OK` |
| `POST`   | `post.html`                          | `201 Created` |
| `HEAD`   | `post.html`                          | `200 Created` *(wrong reason phrase in the code)* |
| `PUT`    | `put.html`                           | `200 Created` *(same)* |
| `DELETE` | — (`DeleteFunction()` is an empty stub) | no response |
| anything else | `unknown.html`                  | `404 Not Found` |

When the requested file cannot be opened, `unknown.html` is served instead — but with the
header of the original method, so a failed GET still returns `200 OK`.

Every response is sent with `Content-Type: text/html; charset=UTF-8`, whatever the actual
file type is.

## Repository layout

```
main.c              server: socket, thread pool, method routing, sendResponse()
utils/myqueue.{c,h} linked-list queue of pending client sockets (enqueue/dequeue)
utils/http.{c,h}    http_request struct, fill_request(), per-method stubs
utils/Consts.h      empty
client/client.c     interactive TCP test client (sends a line, prints the response)
client/client.rb    Ruby test client (requests client/files/<ARGV[0]>.c)
client/files/       document root (1.html, post.html, put.html, unknown.html, 3.c … 10.c)
```

## Build and run

Linux only: the code uses `<linux/limits.h>` and `_GNU_SOURCE`.

```bash
make                 # gcc main.c utils/myqueue.c utils/http.c -o out -lpthread
./out                # listens on port 12345
```

Paths are resolved **relative to the current working directory** (`client/files/…`), so the
server must be started from the repository root.

### Testing

```bash
curl http://localhost:12345/1.html
curl -X POST http://localhost:12345/
curl -X PUT  http://localhost:12345/

# or with the bundled clients
gcc client/client.c -o client/out && ./client/out
ruby client/client.rb 3          # requests client/files/3.c
```

## Known limitations

This is a learning exercise, not a production server. As it stands:

- **no path safety**: the URL is concatenated as-is onto the `client/files/` prefix, with no
  directory-traversal check;
- `strcat()` is called on local arrays sized to the prefix itself
  (`char path[] = "client/files/"`), which overflows;
- `DELETE` returns nothing, and `getFonction()`/`PostFunction()`/`PutFunction()`/
  `HeadFunction()` in `utils/http.c` are unused stubs;
- `httprq` is a **shared global**: concurrent requests overwrite each other;
- request headers and bodies are never parsed (no `Content-Length`, no HTTP version, and none
  of the `http_request` fields besides method and URL);
- files larger than 4096 bytes are sent in chunks with the header length re-counted on every
  `sendResponse()` iteration;
- `close(client_socket)` is called twice (in `sendResponse()`, then in `handle_connection()`);
- the `accept()` loop runs forever, with no clean shutdown and no thread teardown.

## License

MIT — see [LICENSE](LICENSE).
