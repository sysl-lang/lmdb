# lmdb

The [Lightning Memory-Mapped Database](https://www.symas.com/lmdb) for
[sysl](https://github.com/sysl-lang/sysl), bound to the library the machine has.

LMDB is an ordered key-value store in a memory-mapped file: a B+tree with copy-on-write pages and
multi-version concurrency control. Readers never block writers and writers never block readers,
there is exactly one writer at a time, and **a read is a pointer into the map rather than a copy**.
It is what OpenLDAP, Monero and a good deal else keep their data in.

```
sh/sysl/lmdb/
    lmdb.sysl       the binding
    tests.sysl      31 tests against a real LMDB
    c/
        c.sysl      LMDB as C declares it, and the numbers its header computes
package.hocon       who this package is, and what it needs of the machine
```

**There is no shim** — no `c/shim.c`, no C of any kind in this repository. LMDB turns out to be an
unusually clean binding target, and *The boundary* below says why.

## Using it

```hocon
dependencies {
  lmdb { git = "github.com/sysl-lang/lmdb", version = "0.1.0" }
}
```

LMDB has to be installed, and it is everywhere:

```
brew install lmdb              # macOS
sudo apt install liblmdb-dev   # Debian / Ubuntu
sudo pacman -S lmdb            # Arch
```

**pkg-config finds it on all three**, so no flags are needed. That is worth a note, because LMDB's
own Makefile installs no `.pc` file — Homebrew's formula *writes one*, and Debian and Arch ship one
in their `-dev` packages. A machine that has LMDB from source and not from a package manager needs
`--include-path lmdb=<prefix>/include --link-path <prefix>/lib` instead.

## The surface

```sysl
import sh.sysl.lmdb.{env, Channels}
import sysl.text.{Search, from_utf8}

val store = env("/var/db/things", map_size = 1 << 30, max_dbs = 4)?

// Writing. One writer at a time, across the whole environment and across processes.
val w = store.write()?
val db = w.db(Some("users"))?

w.put(db, "u:1".view(), "ed".view())?
w.commit()?

// Reading. Takes no lock and blocks nothing.
val r = store.read()?
val rdb = r.db(Some("users"), create = false)?

print(from_utf8(r.get(rdb, "u:1".view())?)?)

// Walking a range, which is what a cursor is for.
val cur = r.cursor(rdb)?
var at = cur.seek_range("u:".view())?

loop
    at match
        None -> break
        Some(entry) ->
            if !from_utf8(entry.key)?.starts_with("u:") then break
            at = cur.next()?
```

`find` is `get` answering `None` instead of `Err(NotFound)`, which is usually what a caller wants.
`has`, `delete`, `delete_value`, `clear`, `destroy` and `stat` fill in the rest, and `dup_sort`
databases keep several values under one key with `next_value` and `count` to walk them.

## Three things to understand before using it

### 1. A read hands you a pointer into the file

`get` does not copy. What comes back is a `[]const u8` pointing into the memory map, valid **only
until the transaction that produced it ends**:

```sysl
val v = r.get(db, "k".view())?
print(from_utf8(v)?)      // fine — still inside the transaction
r.commit()?
print(from_utf8(v)?)      // NOT fine — the pages may have been reused
```

Nothing in sysl enforces this: the lifetime belongs to LMDB's transaction rather than to any sysl
value, and a binding that copied every read to be safe would give up the single biggest reason to
use LMDB. **Anything wanted past the transaction is copied out inside it.**

### 2. A transaction that is merely dropped aborts

That is the safe way round. An early `?` out of the middle of a sequence of writes leaves the
database untouched rather than half-written, and a caller who meant to keep the work has already
said so with `commit`.

### 3. `map_size` is a hard ceiling, not a hint

LMDB reserves that much address space and never grows it; a write past it fails with `MapFull`. It
costs nothing until used — the map is sparse — so it is normal to name a figure far larger than the
data. The default here is deliberately small enough to be wrong for anything real.

## The boundary — why there is no shim

`15 §7` names three shapes only C can reach: a caller-allocated opaque type, a macro, and a shape
with no sysl spelling. LMDB has none of them.

- **Every handle is allocated by the library.** `mdb_env_create` and `mdb_txn_begin` hand back a
  pointer through an out-parameter, so nothing is placed in storage this side owns and there is no
  size to know. `MDB_env`, `MDB_txn` and `MDB_cursor` are `opaque struct`s here, which says exactly
  that.
- **Every flag and error code is a `#define`**, which is what a `c const` block reads. They are
  asked for by name rather than transcribed.
- **`MDB_val` is two words** — a length and a pointer — and crosses by address everywhere.

**One trap, and it is asserted rather than trusted**: `MDB_val` is `{ size, data }`, the length
first. That is the opposite of how nearly every other C library writes a counted buffer, and getting
it backwards compiles, links, and reads a length as a pointer. `tests.sysl` asserts both offsets
against the C, along with `MDB_stat` and `MDB_envinfo`.

**The `MDB_dbi` looks like a handle and is not.** It is an `unsigned int` index into the
environment's table, so it copies freely and outlives the transaction that opened it. `Db` has no
destructor: `mdb_dbi_close` is documented as unnecessary and unsafe to call while another thread is
using the handle, and LMDB intends a handle to live for as long as the process.

## Two things that cost real time, written down so they cost nobody else any

### A method cannot reach the box it is inside

`Env`, `Txn` and `Cursor` are each **two types** — a private `…Handle` that owns the C pointer and
has the destructor, and a plain copyable value holding a `&` to one. That reads like ceremony and is
forced.

A `&Env` method receives `self` by value, so there is no way for `read()` to put *the box it was
called on* into the transaction it is building. The obvious move — construct a fresh `Env` around
the same pointer — compiles, links, and hands out a **second owner** of one `MDB_env`: two boxes,
two counts, two calls to `mdb_env_close`. This binding was written that way first, and every test
that opened a transaction died with a `SIGTRAP` inside LMDB while the four that never opened one
passed.

Splitting the ownership out fixes it exactly, and the `&` fields then buy the nesting for free: a
`TxnHandle` holds a `&EnvHandle` and a `CursorHandle` holds a `&TxnHandle`, so an environment cannot
be closed while a transaction is open and a transaction cannot end while a cursor is live. The three
closes are ordered by construction rather than by a comment asking for it.

### Ending a transaction with a cursor open: a crash, then a leak

LMDB frees the transaction structure when a transaction ends, and `mdb_cursor_close` reads the
transaction out of the cursor. **So a cursor closed after its transaction has ended is a
use-after-free** — a segfault at program exit, a long way from the `abort` that caused it.

The obvious repair — skip the close once the transaction has ended — trades the crash for a **leak**,
and only in the read case, which is what makes it convincing for a while. A write transaction's
cursors really are freed by LMDB; a read transaction's are not tracked by LMDB at all and are the
caller's to close. Measured: 200,000 rounds of open-cursor-then-abort grew the resident set from
2.8 MB to **89 MB**.

Neither is right. What is right is to close **before**: a transaction tracks its open cursors and
ends them first, and a cursor destroyed earlier takes itself off the list. A caller may now commit
or abort with cursors live, or drop everything in any order.

| rounds of write-commit-read-cursor-abort | peak resident set |
|---|---|
| 2,000 | 1.98 MB |
| 200,000 | 2.05 MB |

## Testing

```
sysl test .
```

**31 passed, 0 failed**, against LMDB 1.0.1.

The two claims worth naming, because they are about LMDB rather than about this binding:

- **A read transaction sees a snapshot.** It reads the database as it was when it began, and a
  commit afterwards does not disturb it — while a reader started later does see the write, which is
  what makes it isolation rather than a stale cache.
- **Without `no_tls` a thread may hold only one read transaction.** The second is refused with
  `BadRslot`, whose message — *"Invalid reuse of reader locktable slot"* — reads as a damaged lock
  file rather than the deliberate design it is. `no_tls = true` moves the slot onto the transaction,
  which is what a program with an event loop or a thread pool wants.

**One number was measured rather than assumed and is worth repeating: the maximum key size here is
8122, not 511.** 511 is what every LMDB 0.9 document quotes; the limit comes from the page size, and
a 1.0 build answers differently. Ask `max_key_size()`. The first version of that test hard-coded the
old figure and walked off the end of its own buffer.

The reader-slot test is the one leak check that is a real test rather than a measurement — LMDB
counts reader slots itself, so a transaction that failed to end is something the *other end* can
see. It was checked by disabling the destructor on purpose: it fails.

### AddressSanitizer

```
SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .
```

**31 passed, 0 failed, instrumentation confirmed** — 22 `__asan_*` undefined symbols including
`__asan_memcpy`.

**Say what that covered, because it is less than it looks.** LMDB is resolved through `pkg_config`,
so its objects were compiled by whoever built the package months ago and no flag of ours instruments
them. What ASan covers here is **the sysl half**: the `MDB_val` handling, the raw-pointer paths, and
the slices handed back out of the map. That is the right half to watch — it is the half this
repository wrote — but it is not a statement about LMDB's B+tree.

> A green ASan run over an unchanged tree proves nothing: `sysl test` caches its artifact and the key
> does not include `SYSL_EXTRA_CFLAGS`, so the flag is silently ignored and the uninstrumented binary
> reused. Check `nm -u <binary> | grep -c asan` before believing one. Filed as card 0415.

## Not bound yet

- **Custom comparators.** `mdb_set_compare` and `mdb_set_dupsort` take a callback, which would be
  ordinary here — every parameter is a pointer, so `&f` works with no `@export`. But a comparator
  that disagrees between two openings of the same database corrupts it silently, and that is a sharp
  enough edge to want a design rather than a wrapper. The default ordering is bytewise, and
  `integer_key` covers the common alternative.
- **`MDB_RESERVE` and `MDB_MULTIPLE`.** Both hand back storage to write into rather than taking
  storage to read — a different shape from everything here.
- **Nested transactions.** `mdb_txn_begin` takes a parent; this binding always passes null.

## Licence

The binding is ISC. LMDB itself is not carried here — it is the library the machine has, under the
OpenLDAP Public License.
