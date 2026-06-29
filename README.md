# lumen-feeds

The Feeds app for **AspisOS**, a capability-based, no-ambient-authority
x86-64 operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

feeds is a standalone RSS/Atom reader: a three-pane desktop client that
subscribes to feeds, fetches them over the network, parses RSS 2.0 / RSS 1.0
(RDF) and Atom, and renders word-wrapped articles. It is a leaf component of the
Lumen desktop, distributed as a [herald](https://github.com/AspisOS/AspisOS)
package, and runs as an **external client** of the
[lumen](https://github.com/AspisOS/lumen) compositor — it connects to
`/run/lumen.sock` over the Lumen window protocol and is handed a shared-memory
buffer to draw into, rather than being an in-process compositor built-in.

## Where feeds fits

AspisOS is decomposed into independent repositories. feeds sits at the leaf of
the graphical stack:

| Repo | Role |
|------|------|
| [`AspisOS/Aegis`](https://github.com/AspisOS/Aegis) | The kernel: capability model, `AF_UNIX` sockets, `memfd`, the `fork`/`execve` and file syscalls the reader runs on. |
| [`AspisOS/lumen`](https://github.com/AspisOS/lumen) | The compositor / display server. Owns the framebuffer; every GUI app is one of its clients. |
| [`AspisOS/glyph`](https://github.com/AspisOS/glyph) | The GUI toolkit feeds links against: the software renderer (`draw_*`, `font_*`), theme/accent values, and the client side of the Lumen protocol (`lumen_client.h`). |
| `AspisOS/lumen-feeds` | **This repo.** The Feeds reader app. |

## What it does

feeds is one app built from five modules, each a separate `.c` compiled against
the shared contract in `src/feeds.h` (the model is fixed-size, NUL-terminated
buffers — `feed_db_t` is `malloc`'d once at ~0.6 MB, no dynamic strings):

- **Three-pane GUI** (`src/main.c`) — opens a **1024x680** window titled "Feeds"
  via `lumen_window_create` (clamped down to the framebuffer on small displays)
  and lays out a Feeds list, an Articles list, and a Reader pane, with a top bar
  showing the unread count and key hints. `layout_panes()` keeps the reader at
  least `P3_MIN` px, stealing width from the other panes on narrow screens.
- **Network fetch** (`src/feeds_net.c`, `feed_fetch`) — downloads happen by
  `fork`/`execve` of `/bin/curl`, **not** in-process: curl carries the
  `NET_SOCKET` capability and the app's own network surface stays in curl. It
  execs curl with an argv array (no shell, so a URL can't become a command):
  `-s -f -L -k -m 25 -A Feeds/1.0 -o <tmp> <url>` — silent, fail on HTTP >= 400,
  follow redirects, skip TLS validation (AspisOS curl ships no CA bundle), cap
  at 25s, and a `Feeds/1.0` User-Agent (some servers 403 a blank UA). The reply
  is fetched to a pid-unique `/tmp/feeds-<pid>.xml`, read back into a freshly
  `malloc`'d NUL-terminated buffer (capped at 1 MiB), and the temp file is always
  unlinked before returning.
- **RSS + Atom parsing** (`src/feeds_parse.c`, `feed_parse`) — a hand-written,
  reentrant, no-XML-library parser that eats arbitrary bytes off the internet
  under three hard invariants: every pointer stays inside `[buf, end)`, every
  copy is bounded and NUL-terminated, and every loop makes forward progress and
  is capped at `ITEMS_MAX`. It detects the root tag (`<rss>` / `<rdf>` / `<feed>`),
  matches tags by **local name** (so `atom:title`, `dc:date`, `content:encoded`
  all resolve), and handles CDATA, attributes, comments and self-closing tags.
  Per item it extracts title, link, date and summary, mapping the right fields
  for each format (Atom `<content>`/`<summary>`, `<updated>`/`<published>`, and
  `rel`-aware `<link href>` selection; RSS `<description>`/`content:encoded`,
  `<pubDate>`/`dc:date`, `<link>` text or RDF `rdf:about`).
- **HTML -> plain text** (`src/feeds_text.c`, `html_to_text`) — strips tags,
  decodes the common named and numeric entities (`&amp;` `&lt;` `&gt;` `&quot;`
  `&#39;` `&#NNN;` `&nbsp;` `&mdash;` `&hellip;` ...), and collapses whitespace
  runs to single spaces, so the model holds clean plain text. The parser runs
  every title and summary through it.
- **Persistence** (`src/feeds_store.c`, `feed_db_*`) — subscriptions live in
  `$HOME/.feeds` (falling back to `/root/.feeds`), one `<url>\t<title>` line each,
  saved atomically via a `.feeds.tmp` + `rename()`. First run with no file seeds
  a few default subscriptions (AspisOS News, Hacker News, LWN). Read-state is a
  separate `$HOME/.feeds-read` cache of read article links, appended on open and
  re-applied after every re-parse so already-read items don't resurface.

The reader does its own greedy pixel word-wrap (`wrap_body`), measuring with
`font_text_width` and breaking over-long words character-by-character so nothing
clips. It is integer-only throughout — the Aegis FPU is fragile, so all text
measurement and scrolling is in pixels/lines. Input is keyboard and mouse:
Tab cycles panes, arrows/`j`/`k` move selection or scroll, Enter loads/opens,
`r`/`R` refresh one feed or all, `a` adds a subscription (inline URL field),
`d` deletes the selected feed, `q`/Esc quits; the mouse wheel pages the focused
pane.

## Capabilities

AspisOS grants a process no ambient authority; it can touch the system only
through capabilities declared for it at exec time. feeds's policy
(`pkg/etc/aegis/caps.d/feeds`) is:

```
service NET_SOCKET
```

feeds is the one app in this batch of peeled components that touches the
network, so on top of the baseline `service` profile its policy grants the
**`NET_SOCKET`** capability. That capability is what lets the app reach the
network — but note the design keeps the actual socket surface in `curl`, which
has its own `NET_SOCKET` grant: `feed_fetch` `fork`/`execve`s `/bin/curl` rather
than opening sockets itself. The app still needs `NET_SOCKET` to spawn and wait
on a child that performs network I/O within the same capability domain. No other
elevated capability is requested — no filesystem caps beyond the `service`
baseline, no setuid, nothing ambient.

## Status

feeds is the most substantial of the freshly peeled Lumen apps: a complete,
working three-pane reader with live fetch, RSS/Atom parsing, persistent
subscriptions and read-state. It is honest about its limits — fixed-size model
(`FEEDS_MAX` subscriptions, `ITEMS_MAX` items per feed), blocking refresh, and
no PageUp/PageDown yet (those are E0 scancodes the PS/2 path doesn't deliver;
the mouse wheel covers paging today). It is expected to grow as AspisOS matures.

## Building

feeds builds with a musl cross-compiler against a **pinned**
[glyph](https://github.com/AspisOS/glyph) toolkit artifact (the GUI libraries it
links), then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `make` runs `tools/fetch-glyph.sh $(GLYPH_VERSION)` to download and unpack the
  pinned toolkit into `toolkit/`, compiles **all** of `src/*.c` into one binary
  (`component.elf`) against it, then packs.
- `MUSL_CC` is the musl cross-compiler (defaults to `musl-gcc` on `PATH`; the
  only toolchain assumption — point it at an Aegis-native `cc` to build on-device
  in the future).
- `HERALD_KEY` is the ECDSA-P256 key that signs the `.hpkg`.
- `GLYPH_VERSION` pins the toolkit release; `VERSION` is this app's own version.

Output: `lumen-feeds.hpkg` (a `class=system` herald package) +
`lumen-feeds.hpkg.sig`.

## Package payload

`lumen-feeds.hpkg` is a **herald `class=system` package**: a manifest-first
uncompressed POSIX `ustar` archive with a detached ECDSA-P256/SHA-256 signature
(`tools/pack.sh`). Its herald id (`lumen-feeds`) deliberately differs from the
bundle/exec name (`feeds`), and it installs across two trees — which is exactly
why it is `class=system` (first-party, signature-trusted, installed verbatim)
rather than an ordinary single-prefix package:

```
/apps/feeds/feeds            the app binary
/apps/feeds/app.ini          the bundle descriptor (name=Feeds, exec=feeds)
/etc/aegis/caps.d/feeds      its capability policy (service NET_SOCKET)
```

## Repository layout

```
src/        Feeds source: feeds.h (shared model + module contract),
            main.c (Lumen three-pane GUI), feeds_net.c (fork/exec curl fetch),
            feeds_parse.c (RSS 2.0 / RDF / Atom parser), feeds_store.c
            (subscriptions + read-state), feeds_text.c (HTML -> plain text)
pkg/        install-tree skeleton shipped verbatim (apps bundle + caps.d)
tools/      fetch-glyph.sh (pinned toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build (compile every src/*.c into one binary) -> pack
sample-feed.xml  a test-fixture feed for local parser checks (NOT shipped in the package)
VERSION         this app's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — feeds is an external client of the compositor, so installing
it pulls [lumen](https://github.com/AspisOS/lumen). lumen also ships the desktop
fonts (Inter, JetBrains Mono), so feeds inherits them transitively; there is no
separate font package. At runtime feeds also expects `/bin/curl` (with its own
`NET_SOCKET` capability) on the system to perform fetches.
