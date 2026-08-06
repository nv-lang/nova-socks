[English](README.md) | **Русский**

# nova-socks

SOCKS5-клиент для [Nova](https://nv-lang.org) — [RFC 1928](https://www.rfc-editor.org/rfc/rfc1928)
(SOCKS Protocol Version 5) с аутентификацией по логину/паролю
([RFC 1929](https://www.rfc-editor.org/rfc/rfc1929)). Чистый Nova поверх
эффекта `Net` из `std.net` — без нативного кода, без C-прослойки, без
завендоренных исходников.

Почему отдельный пакет, а не часть `nova-http`: SOCKS5 проксирует ЛЮБОЕ
TCP-соединение, не только HTTP — положить его внутрь HTTP-библиотеки
значило бы заставить каждого, кому нужен голый туннель, тащить весь
HTTP-стек. Все основные экосистемы держат это раздельно: Go
`golang.org/x/net/proxy`, Rust `tokio-socks`/опциональная фича `socks` у
reqwest, Python `PySocks`/`requests[socks]`, Node `socks` +
`socks-proxy-agent`.

Один из мотивирующих сценариев (есть и другие): некоторые провайдеры прокси
говорят только по SOCKS5 с логином/паролем, а встроенные настройки прокси в
ОС (например, в Windows) могут принимать только обычный HTTP-прокси без
возможности передать креды SOCKS5 — локальному мосту, который завершает
SOCKS5 и переоткрывает его как обычный HTTP, для этого завершения нужен
SOCKS5-клиент. Этот пакет — именно такой клиент; сам мост он не поставляет.

## Что входит в объём (V1)

- **Только команда CONNECT** — стандартный глагол «открыть TCP-туннель
  через прокси». `BIND` и `UDP ASSOCIATE` не реализованы.
- **Целевые адреса: IPv4-литерал или доменное имя.** IPv4-литерал в виде
  точечной записи (`"93.184.216.34"`) отправляется как `ATYP 0x01`; всё
  остальное — как доменное имя (`ATYP 0x03`, резолвится самим прокси).
- **Целевые адреса IPv6 НЕ поддержаны** — если в ответе прокси приходит
  привязанный адрес IPv6 (`ATYP 0x04`), клиент возвращает типизированный
  `Err(UnsupportedAddressType)`, а не неверно его разбирает. Это осознанная
  граница V1, а не недосмотр.
- **Аутентификация по логину/паролю** (RFC 1929) — предлагается
  автоматически, когда заданы и логин, и пароль; `Err(AuthRequired)`, если
  сервер настаивает на ней, а креды не даны.
- **Не реализовано:** SOCKS5-*сервер* (только клиент), SOCKS4/4a,
  аутентификация GSSAPI (RFC 1961 — на практике редкость), UDP ASSOCIATE.

## Использование

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

`user`/`pass` — `Option[str]`: передайте `None, None` для прокси без
аутентификации.

## Устройство

Каждое сообщение хендшейка (приветствие, выбор метода, запрос/ответ auth,
запрос/ответ CONNECT) — **чистая функция кодирования/декодирования над
`[]u8`**: без эффекта `Net`, поэтому весь проводной протокол проверен
юнит-тестами на байтовых фикстурах, без единого сетевого вызова
(`src/socks5_test.nv`). Сам `socks5_connect` — тонкая обвязка с эффектом
`Net`: прочитать N байт → декодировать → закодировать → записать N байт.
Полный разбор байтовой раскладки (переписанный с текста RFC, не по
памяти) — в `src/socks5.nv`.

## Структура

```
nova-socks/
├── nova.toml              [package] name = "socks"; [lib] src = "src"
└── src/
    ├── socks5.nv             SocksError + socks5_connect (RFC 1928/1929)
    └── socks5_test.nv        peer tests — byte fixtures, same-module
```

## Автономная сборка

Нужен тулчейн Nova (CLI `nova` + clang). У этого пакета вообще нет нативных
зависимостей.

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

**Ручное smoke-тестирование — вне CI**: чтобы проверить реальный
end-to-end туннель, нужен настоящий внешний SOCKS5-прокси, недоступный в
автоматическом прогоне. `src/socks5_test.nv` покрывает проводной протокол
только байтовыми фикстурами — он не заявляет о end-to-end проверке против
живого прокси.

## Лицензия

Двойная лицензия — [MIT](LICENSE-MIT) или [Apache-2.0](LICENSE-APACHE), на
ваш выбор — те же условия, что у компилятора Nova и стандартной библиотеки.
