**English** | [Русский](README.ru.md)

# nova-socks

A SOCKS5 client for [Nova](https://nv-lang.org) — [RFC 1928](https://www.rfc-editor.org/rfc/rfc1928)
(SOCKS Protocol Version 5) with username/password authentication
([RFC 1929](https://www.rfc-editor.org/rfc/rfc1929)). Pure Nova over
`std.net`'s `Net` effect — no native code, no C shim, no vendored source.

Why a standalone package, not part of `nova-http`: SOCKS5 proxies **any**
TCP connection, not just HTTP — bundling it into an HTTP library would force
everyone who just wants a raw tunnel to pull in the whole HTTP stack.
Every mainstream ecosystem keeps them separate: Go `golang.org/x/net/proxy`,
Rust `tokio-socks`/reqwest's optional `socks` feature, Python
`PySocks`/`requests[socks]`, Node `socks` + `socks-proxy-agent`.

One motivating scenario (there are others): some proxy providers only speak
SOCKS5 with a username/password, while an OS's built-in proxy settings (e.g.
Windows) may only accept a plain HTTP proxy with no way to carry SOCKS5
credentials — a local bridge that terminates SOCKS5 and re-exposes it as
plain HTTP needs a SOCKS5 client to do the terminating. This package is
that client; it doesn't ship a bridge itself.

## What's in scope (V1)

- **CONNECT command only** — the standard "open a TCP tunnel through the
  proxy" verb. `BIND` and `UDP ASSOCIATE` are not implemented.
- **Target addresses: IPv4 literal or domain name.** A dotted-quad IPv4
  literal (`"93.184.216.34"`) is sent as `ATYP 0x01`; anything else is sent
  as a domain name (`ATYP 0x03`, resolved by the proxy).
- **IPv6 targets are NOT supported** — if the proxy's reply carries an
  IPv6 bound address (`ATYP 0x04`), the client returns a typed
  `Err(UnsupportedAddressType)` rather than mis-decoding it. This is a
  deliberate V1 boundary, not an oversight.
- **Username/password authentication** (RFC 1929) — offered automatically
  when both a username and a password are given; `Err(AuthRequired)` if the
  server insists on it and none were given.
- **Not implemented:** a SOCKS5 *server* (client only), SOCKS4/4a, GSSAPI
  authentication (RFC 1961 — rare in practice), UDP ASSOCIATE.

## Usage

```nova
import socks.{socks5_connect}
import std.net.{real_net}

fn main() Net -> () {
    with Net = real_net() {
        match socks5_connect(
            "proxy.example.com", 1080,
            Some("alice"), Some("s3cret"),
            "example.com", 80
        ) {
            Ok(consume stream) => {
                // `stream` is a live TCP tunnel to example.com:80,
                // established through the SOCKS5 proxy — read/write it
                // exactly like a direct connection.
                stream.close()
            }
            Err(e) => { /* e.to_str() — typed SocksError */ }
        }
    }
}
```

`user`/`pass` are `Option[str]` — pass `None, None` for an unauthenticated
proxy.

## Bounding the handshake time

`socks5_connect` has **no built-in timeout** — a caller who needs one wraps
the call in `supervised(timeout: …)` (the general Nova idiom for bounding a
blocking `Net` operation):

```nova
with Fail[TimeoutError] = |_e| { /* handle the timeout */ } {
    supervised(timeout: 5000.to_millis()) {
        match socks5_connect("proxy.example.com", 1080, None, None, "example.com", 80) {
            Ok(consume stream) => { /* use it */ }
            Err(e) => { /* handshake failed within the deadline */ }
        }
    }
}
```

**Known caveat, found during a 2026-08-06 audit-review pass (not fixed in
this package):** ad hoc loopback testing showed that `timeout:` reliably
interrupts a blocked `TcpListener.accept()` (this is regression-tested —
`std/src/net/supervised_cancel_accept_test.nv`), but did NOT reliably
interrupt a SECOND `TcpStream.read()` on a connection where the peer sent
part of a protocol field and then closed — exactly the retry pattern
`socks5_connect`'s internal `read_exact_n` uses. This looks like an open
`std.net`/runtime question, not something fixable from this package —
minimal repro:
[`docs/plans/repro/p249_second_read_after_partial_hangs.nv`](https://github.com/nv-lang/nova/blob/main/docs/plans/repro/p249_second_read_after_partial_hangs.nv)
in the `nova` repo. Until it's resolved, a proxy that sends a truncated
field and then goes silent (rather than closing cleanly) may hang past your
`timeout:` deadline.

## Design

Every handshake message (greeting, method selection, auth request/reply,
CONNECT request/reply) is a **pure encode/decode function over `[]u8`** —
no `Net` effect, so the whole wire protocol is unit-tested with byte
fixtures and zero network (`src/socks5_test.nv`). `socks5_connect` itself is
a thin `Net`-effect wrapper: read N bytes → decode → encode → write N
bytes. See `src/socks5.nv` for the full byte-layout commentary (transcribed
from the RFC text, not from memory).

## Layout

```
nova-socks/
├── nova.toml              [package] name = "socks"; [lib] src = "src"
└── src/
    ├── socks5.nv             SocksError + socks5_connect (RFC 1928/1929)
    └── socks5_test.nv        peer tests — byte fixtures, same-module
```

## Building standalone

Requires the Nova toolchain (`nova` CLI + clang). No native dependencies at
all for this package.

```sh
# `nova` does not (yet) bundle/locate the standard library relative to the
# nova.exe install — a standalone package must point it at a Nova checkout's
# std/ via NOVA_STD_PATH:
export NOVA_STD_PATH=/path/to/nova/std

# Boehm GC (mandatory Nova runtime dep) needs its own lib/include dirs —
# point NOVA_GC_LIB_DIR (+ optional NOVA_GC_INCLUDE_DIR) at a prebuilt
# bdwgc if it isn't reachable via the default vcpkg/system lookup.
export NOVA_GC_LIB_DIR=/path/to/vcpkg_installed/x64-windows-static/lib
export NOVA_GC_INCLUDE_DIR=/path/to/vcpkg_installed/x64-windows-static/include

# Ditto for the compiler's own C runtime (compiler-codegen/nova_rt/ + the
# libuv submodule it needs):
export NOVA_CG_INCLUDE=/path/to/nova/compiler-codegen
export NOVA_RT_DIR=/path/to/nova/compiler-codegen/nova_rt

nova test src
```

**Manual smoke-testing is out of CI scope**: verifying an actual end-to-end
tunnel needs a real external SOCKS5 proxy, which isn't available in an
automated run. `src/socks5_test.nv` covers the wire protocol with byte
fixtures only — it does not claim end-to-end testing against a live proxy.

## License

Dual-licensed under [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE), at
your option — same terms as the Nova compiler and standard library.
