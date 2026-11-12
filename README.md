# dream_socket_server

> **This is a fork of [glisten](https://github.com/rawhat/glisten), published by the [Dream](https://github.com/TrustBound/dream) project. If you are not using Dream, use glisten.**
>
> The package is named for what it does, a socket server, so that it does not appear when people search Hex for glisten.

## Why this package exists

glisten is the socket library that mist, and so Dream, runs on. It is written by Alex Manning (rawhat) and its contributors, and it has given a great deal to us and to the rest of the Gleam community. This package is not a replacement for it and not a judgement of it.

We publish this fork for four reasons.

1. **Dream needed a fix that is not released.** glisten set every socket's read buffer to the size of the kernel's receive buffer. Under load that held a large buffer for every read (about 3,000 MB against 155 MB in the same test on macOS, and 139 MB against about 70 MB on Linux) and gave no measurable speed. The fork stops doing that and adds `glisten.with_buffer_size` for anyone who wants a different size. That is two changes, on the branch `socket-buffer-9.0.1`.
2. **Giving changes back takes time.** Each change goes upstream as its own small pull request, in the maintainer's style, and is reviewed and released on the maintainer's schedule. That is the right way to do it, and it cannot be hurried. We intend to send both.
3. **Dream could not wait any longer.** This version of Dream has already been delayed too long, and its users need the fixes now.
4. **Hex and Gleam leave no other way to ship a fixed dependency.** A Hex package may depend only on other Hex packages, so Dream cannot depend on a git fork or a local path. A Gleam package is built only from the real files in its own `src/` folder, so Dream cannot use a git submodule as its source either: we tested it, and the published package comes out with the files in the wrong place. The choices left were to copy glisten's source into Dream, or to publish the fork as its own package. Publishing the fork keeps glisten's code, history and licence in one place, apart from Dream's.

## What you need to know before you use it

- **This package defines the same modules as glisten** (`glisten`, `glisten/...`). A project cannot depend on both `glisten` and `dream_socket_server`. Gleam stops the build with a duplicate module error.
- **The code is glisten 9.0.1 plus our two changes, and nothing else.** We do not rename, reformat or restyle glisten's source. What is published here is what we send upstream.
- **It is temporary.** When a glisten release contains the fix, Dream goes back to `glisten`, and this package gets a final release that says so.
- **Report problems with this package to [TrustBound/glisten](https://github.com/TrustBound/glisten/issues), not to the glisten maintainers.** They did not publish it and do not support it.

## Licence and credit

glisten is licensed under Apache-2.0, and so is this fork. The copyright stays with Alex Manning and the glisten contributors. Our changes are offered under the same licence.

---

*The original glisten README follows.*

---

# glisten

[![Package Version](https://img.shields.io/hexpm/v/glisten)](https://hex.pm/packages/glisten)
[![Hex Docs](https://img.shields.io/badge/hex-docs-ffaff3)](https://hexdocs.pm/glisten/)

A Gleam TCP server library.  Built on top of `gleam_otp`, it provides a
supervisor over a pool of socket acceptors.  Each acceptor will block on `accept`
until a connection is opened.  The acceptor will then spawn a handler process
and then block again on `accept`.

Below is a simple example that will echo received messages.  You can get this
up and running with the following:

```sh
$ gleam new <your_project>
$ cd <your_project>
$ gleam add glisten gleam_erlang gleam_otp
```

Then place this code in `src/<your_project>.gleam`:

```gleam
import gleam/bytes_tree
import gleam/erlang/process
import gleam/option.{None}
import glisten.{Packet}

pub fn main() {
  let assert Ok(_) =
    glisten.new(fn(_conn) { #(Nil, None) }, fn(state, msg, conn) {
      let assert Packet(msg) = msg
      let assert Ok(_) = glisten.send(conn, bytes_tree.from_bit_array(msg))
      glisten.continue(state)
    })
    // NOTE:  By default, `glisten` will listen on the loopback interface.  If
    // you want to listen on all interfaces, pass the following.  You can also
    // specify other interface values, including IPv6 addresses.
    // |> glisten.bind("0.0.0.0")
    |> glisten.start(3000)

  process.sleep_forever()
}
```

TLS is also handled using the `glisten.with_tls` method.  This requires a
certificate and key file path.  The rest of the handler flow remains unchanged.

`glisten` doesn't provide a public API for connected clients.  In order to hook
into the socket lifecycle, you can establish some functions which are called
for the opening and closing of the socket.  An example is provided below.

To serve over TLS:

```gleam
// ...
pub fn main() {
  let assert Ok(_) =
    glisten.new(
      // omitted
    )
    |> glisten.with_tls(
      certfile: "/path/to/server.crt",
      keyfile: "/path/to/server.key",
    )
    |> glisten.start(port: 8080)
  process.sleep_forever()
}
```

But you can also drop down to the lower level listen/accept flow if you'd prefer
to manage connections yourself, or only handle a small number at a time.

```gleam
import gleam/result
import glisten/socket/options.{ActiveMode, Passive}
import glisten/tcp

pub fn main() {
  use listener <- result.try(tcp.listen(8000, [ActiveMode(Passive)]))
  use socket <- result.try(tcp.accept(listener))
  use msg <- result.try(tcp.receive(socket, 0))
  echo #("got a msg", msg)

  Ok(Nil)
}
```

See [mist](https://github.com/rawhat/mist) for HTTP support built on top of this library.
