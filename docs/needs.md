# What spool needs from twill

spool is written in twill and runs. This file was the reason it did not: it is
the list of language and runtime features the source uses that `mode systems`
did not provide, with the file that needs each one and what spool did in the
meantime.

Most of it is now a record rather than a queue. Of the fourteen entries below,
thirteen are delivered as of twill 1.7.1, which is the release `spool.toml` and
CI pin. **One is still open, and it is entry 1: there is no process interface.**
That is verified rather than assumed: no builtin in twill 1.7.1 starts a
subprocess, `src/vendor.tw` calls `run`, and any command that reaches git dies
with `undefined variable "run"`. Vendoring from a git source is the one spool
feature that waits on it.

It was meant to be read as a work queue for the language, not as a complaint.
Every entry was reached by writing real code and hitting the wall, which is the
only way this list is worth anything. Each entry keeps what it said while it was
open, because the ugliness of a workaround is information about how badly the
feature was wanted, and because a queue that deletes its delivered items leaves
no record of whether asking worked.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64` with bitwise operations and defined wrapping, `Str` with
length, byte indexing and slicing, `Arr[T]`, `Dict[Str, V]` with
insertion-ordered iteration, `struct`, and `read_file`. Everything spool uses
beyond that is below.

## Was blocking: spool could not work at all without these

Entry 1 still is.

### 1. A process interface

**Needs:** `run(program: Str, argv: Arr[Str], dir: Str) -> Res[Str, Str]`
**Used by:** `src/vendor.tw`
**Status:** **still open on twill 1.7.1.** The only entry here that is.

The `Res` in the signature is the one change since this was written. The
`"!"`-flag encoding entry 10 describes is gone everywhere else in spool, and a
process interface should not be the last place it survives.

This is the largest gap and it is not a small one. spool fetches packages by
running `git clone`, `git fetch`, `git tag`, `git rev-list`, `git show` and
`git checkout`. Shelling out to git is a deliberate design choice rather than a
shortcut: it borrows the user's existing credentials, proxies and host keys, and
it is what lets a package manager avoid taking on a network stack. Nothing in
`docs/self-hosting.md` provides it, and section 1.2 explicitly stops at "no
sockets", so neither route out is currently open.

Either a process interface or an HTTPS client has to exist before spool can
fetch anything. A process interface is the smaller ask and the more useful one,
and it is what the self-hosted toolchain will want anyway for driving a linker.

The security note belongs here too: this widens what running a `.tw` file can
do more than any other item on this list, and it should be a considered decision
rather than a side effect of wanting a package manager.

### 2. Writing files

**Needs:** `write_file(path: Str, contents: Str) -> Str`
**Used by:** `src/commands.tw` (the lockfile, the manifest, the vendor README)
**Status:** **done** (twill 1.7). `write_file(path, contents) -> Res[Unit, Str]`.

A package manager that cannot write a lockfile is a linter. `spool init` and
`spool install` both write files now: `twill run main.tw install` in an empty
project produces a `spool.lock` and a `twill_modules/README`.

twill also grew `write_text_or(path, contents) -> Bool` and
`read_text_or(path, fallback) -> Str` for the caller who has already decided
what a failure means. spool does not use either. Every write it makes is one a
user asked for, and a write that silently did not happen is the failure mode a
package manager can least afford.

### 3. Directory operations

**Needs:** `list_dir(path) -> Arr[Str]`, `is_dir(path) -> Bool`,
`path_exists(path) -> Bool`, `mkdir_all(path)`, `remove_all(path)`
**Used by:** `src/vendor.tw`, `src/commands.tw`
**Status:** **done** (twill 1.7), with two spellings to note.

`is_dir` is `path_is_dir`. `list_dir` answers `Res[Arr[Str], Str]` rather than a
bare `Arr[Str]`, which is the right answer and is also a trap: `src/vendor.tw`
and `src/commands.tw` both indexed the `Res` directly, so `walk` and `prune`
were broken in the way entry 10 describes. Both are fixed.

twill 1.7.1 delivers more than this entry asked for: `remove_file`,
`remove_dir`, `rename`, `temp_dir`, `file_size`, `mtime`, `read_file_at`, and
the `path_join` / `path_base` / `path_dir` / `path_ext` / `path_stem` /
`path_normalize` / `path_is_abs` family. `path_join` emitting a forward slash on
every platform is what deleted spool's hand-rolled `join_path`, and it is what
`read_tree` depends on for a package hash that is the same on Windows and on
Linux.

The content hash covers every file in a vendored package, so spool has to be
able to enumerate one. `remove_all` is needed because a package that fails
verification has to be replaced rather than merged over, and `path_exists` is
needed to tell "not installed yet" from "installed and wrong", which are
different messages.

`list_dir` returning entries in a defined order would be a bonus but is not
required: `src/vendor.tw` sorts them, because the package hash must not depend
on filesystem order.

### 4. Process arguments, output and exit

**Needs:** `args() -> Arr[Str]`, `cwd() -> Str`, `write_err(Str)`, `exit(I64)`,
and `print` accepting a `Str`
**Used by:** `main.tw`
**Status:** **done** (twill 1.7). All five, plus `env`.

`cwd` was the one addition and it landed as `Res[Str, Str]`. `main.tw` bound the
`Res` itself for a while and handed it to `path_join`, so every command failed
before doing anything; that is bug 1 in entry 10.

## Was blocking: the language features the source assumes

All four delivered.

### 5. `Str` concatenation with `+`

**Used by:** every file.
**Status:** **done** (twill 1.7). `+` concatenates in systems mode, and the
whole suite passing is the demonstration.

The related ask is delivered too, under another name: `chr(b)` returns the
one-byte `Str` for a byte value, so `str_from_byte` is not needed. The `HEX`
table in `src/strutil.tw` stays anyway, because what it does is a
nibble-to-digit lookup rather than a byte-to-string conversion, and `HEX[n]` is
clearer than `chr` plus a branch on whether the nibble is a digit or a letter.

Where `+` is the wrong tool it is now avoidable: `src/lockfile.tw` and
`src/pkghash.tw` build into a `Bytes` with `std/text`'s `push_str`, because both
concatenate in a loop over a whole dependency tree and `+` there is quadratic in
the size of the output.

### 6. The spelling of the I64 bitwise operators

**Used by:** `src/strutil.tw`
**Status:** **done** (twill 1.7), and the question this entry asked is the one
the answer turned on.

They are different operators with different names. `docs/language-guide.md` in
the twill repository spells the bitwise ones `band`, `bor`, `xor`, `shl`, `shr`,
usable infix or as calls, with `bnot(a)` for the complement. `and` and `or` stay
boolean. `src/strutil.tw` writes `band` today, which is the whole of what spool
needed from this.

`src/sha256.tw` is gone, so the file that would have exercised these hardest is
no longer here; see entry 13.

### 7. `Bool` as a declared type

**Used by:** every file.
**Status:** **done** (twill 1.7). It was an oversight in the document rather
than a missing feature, as this entry guessed. `-> Bool` checks and runs, in the
dozen places spool writes it.

### 8. `continue` in loops

**Used by:** `src/toml.tw`, `src/manifest.tw`, `src/lockfile.tw`,
`src/resolve.tw`, `src/commands.tw`
**Status:** **done** (twill 1.7). `continue` works, including inside a `match`
arm, which is what `src/vendor.tw`'s catalog walk needs.

The parsers are written as a loop over lines with an early `continue` per line
shape. Without it every parser here grows a level of nesting per case, which is
exactly the readability the self-hosting document is trying to buy.

## Not blocking, but the source was worse without them

All six delivered, two of them only in the language: entries 9 and 11 are refactors this repository has
not done yet.

### 9. Sum types and `match`

**Would improve:** `src/toml.tw`, `src/semver.tw`, `src/manifest.tw`
**Status:** **done in the language** (twill 1.7), half taken up in spool.

`src/semver.tw` is converted. `enum ConstraintKind { Exact(Version),
Caret(Version), Any }` replaced the `I64` tag beside a `base` that meant nothing
in the `Any` case, and `matches` is an exhaustive `match` rather than a chain of
`if`s with a fall-through into the caret arm.

`src/toml.tw`'s `Pair` is not converted. It still carries a `value: Str`, a
`fields: Dict[Str, Str]` and an `is_table: Bool` saying which one is real, which
is a two-case sum written out by hand with nothing stopping a caller from
reading the wrong field. That is a refactor this repository owes itself rather
than something the language withholds.

### 10. `Res[T, E]`, `Opt[T]`, and returning two values

**Would improve:** every file.
**Status:** done, both halves (2026-08, on twill 1.7). Neither convention
survives anywhere in spool.

**Done.** `Doc`, `Manifest`, `Lock`, `Resolution`, `Project` and `vendor.Fetched`
have lost their `err` fields, and every function that returned a `Str` that was empty on
success returns a `Res` instead: `toml.parse`, `manifest.parse`,
`validate_name`, `validate_entry`, `lockfile.parse`, `resolve.resolve`,
`catalog_deps`, `pkghash.verify`, `commands.open`, `load_lock`,
`lock_satisfies`, `materialise`, all five `cmd_*`, and `vendor.build_catalog`,
`export` and `read_tree`. `main.check` takes the `Res`, so there is no message to take the
length of and no way to pass one that was never read.

**Seven bugs this found, none of which the tests could see**, because the suite
calls the command functions with fixture strings and never goes through `main`
or touches a file. Three came out of the conversion itself:

1. `main` bound `cwd()` -- a `Res[Str, Str]` -- and handed it to `path_join`, so
   every command died with "path_join expects strings" before doing anything.
   `spool init` has been broken since `cwd` gained that return type.
2. `open` and `load_lock` passed `read_file(path)` -- also a `Res` -- straight
   into a parser, which took the length of it. Every command that opens a
   project failed the same way.
3. `main.tw` ended with a `main()` call. `twill run` executes a systems-mode
   file's top level and *then* calls `main()` itself, which it has done since
   twill 1.6.1, so the whole program ran twice: `spool init` wrote the manifest
   and then reported that it already existed, and exited 1.

Four more were the same mistake made against builtins that had gained a `Res`
rather than against spool's own functions, and they survived the first pass. A
second reading of every call site found them:

4. `cmd_add` took the length of `validate_name`'s `Res[Unit, Str]`, so `spool
   add` died with "len expects a tensor, list, string, dict or bytes" for every
   name, valid or not, after printing `added`.
5. `materialise` did the same to `vendor.export` and to `pkghash.verify`, so a
   failed export and a failed integrity check would both have been read as
   success.
6. `vendor.read_tree` pushed `read_file`'s `Res` straight into the array of file
   contents it hashes. An unreadable file would have gone into the package hash
   as a `Res` value rather than being reported.
7. `vendor.walk` and `commands.prune` indexed `list_dir`'s `Res` as an array.

All seven are the same shape: a value that says whether it worked, used as
though it could not have failed. The pattern that catches them is not a test, it
is `?` and `match` at the boundary, which is what this entry asked for.

**The `"!"`-flag convention is gone too**, which this entry rightly called the
worse of the two: a `Str` whose first byte said whether the rest was a value or
a message. `toml.unquote` and `manifest.remove_dep` are `Opt[Str]` -- neither
had a message to carry, only "there isn't one" -- and `vendor.git`,
`ensure_clone`, `commit_for` and `read_manifest_at` are `Res[Str, Str]`.
`tag_for` is `Opt[Str]`.

`git_ok` and `git_out` are deleted. They existed only to make the flag
survivable, and every one of the sixteen places that called them is now a
`match` or a `?`. `run`'s own flag-byte encoding is unwrapped exactly once, in
`git`, which is the only function that calls it -- so the convention stops at
the boundary instead of leaking through the package.

What that removes is a class of silent wrongness, not lines: `unquote_value`
without `unquote_ok` gave a value with a leading space; `remove_dep`'s result
written without slicing gave a manifest with one. Both compiled.

*What the entry said while it was open:*

Functions that can fail return one of:

- a struct with an `err: Str` field that is empty on success
  (`Manifest`, `Lock`, `Resolution`, `Doc`), or
- a `Str` whose **first byte is a status flag**: `" "` for success with the
  value following, `"!"` for failure with the message following
  (`toml.unquote`, `manifest.remove_dep`, `vendor.git`, `vendor.commit_for`).

The second is genuinely bad. It is unchecked, it is invisible at the call site,
and `git_ok` / `git_out` accessors exist only to make it survivable. Every use
of it in this source is a place `Res[T, E]` and `?` would delete a bug class.

A narrower fix that would also work: any way at all to return two values, or a
generic two-field struct. The subset has no generics over user structs, so
`Result[Str]` cannot be written even by hand.

### 11. `Dict` values that are not scalars

**Would improve:** `src/resolve.tw`
**Status:** **done** (twill 1.7). It was the documentation request. `V` may be
any type: `Dict[Str, Arr[Str]]` and a `Dict` whose values are structs both check
and run.

`resolve.Catalog` still holds `Dict[Str, Str]` where the values are
**comma-separated version lists** and **whole rendered manifests**, re-parsed on
every read. It wants `Dict[Str, Arr[Version]]` and `Dict[Str, Manifest]`, and
nothing is stopping it any more. Like entry 9, this is a refactor spool owes
itself.

### 12. A sort, or a comparison-function parameter

**Would improve:** `src/strutil.tw`, `src/manifest.tw`, `src/lockfile.tw`,
`src/resolve.tw`
**Status:** **half delivered** (twill 1.7). The half that was a language
question is answered: a function may be passed to a systems-mode function, with
a `fn(Str, Str) -> Bool` parameter type, and it checks and runs.

There is still no generic sort builtin, and there are still four near-identical
insertion sorts in this source, one per element type, differing only in the
comparison. They are correct and small, and having four of them is still four
times as many places for the ordering that the lockfile depends on to go wrong.
A comparison-taking sort can be written in twill now. Nobody has written it.

### 13. A `sha256` builtin

**Would improve:** `src/sha256.tw`
**Status:** **done** (twill 1.7), and this entry argued against it.

The argument was that a hash spool needs is spool's to own: a package manager
that does not verify what it fetched is a supply-chain hole, a non-cryptographic
hash would detect corruption but not substitution, and substitution is the
threat.

What settled it the other way is that a digest the whole toolchain has to agree
on byte for byte should have one implementation and not one per repository.
`warp` wanted the same function for cache keys, and two transcriptions of
SHA-256 that must agree is the worse risk.

It is `std/hash` now. `src/sha256.tw` is deleted, `src/pkghash.tw` calls
`sha.hash_str`, and `tests/sha256_test.tw` still checks the published vectors,
including the padding boundaries at 55, 56 and 64 bytes, through `std/hash`. The
part that stayed here is the part that is actually spool's: the canonical,
length-prefixed serialisation of a file tree in `src/pkghash.tw`.

### 14. Mandatory-typing policy questions the source ran into

**Status:** **answered** (twill 1.7). All four assumptions below were the right
ones, and the demonstration is that this source checks and its suites pass. They
are left written down because they are the questions the next program written in
this subset will ask.

- Are `Arr` and `Dict` literals (`[]`, `{}`) inferred from the annotation on the
  binding? spool writes `let out: Arr[Str] = []` and assumes yes. The `{}`
  literal is the awkward one, since `{` already means a block or a record.
- Is a struct field's type enough to type a struct literal's field, or must
  every literal be annotated? spool assumes the former.
- May a function take a struct and mutate it, with the mutation visible to the
  caller, when the struct was constructed by the caller? Section 1.2 says
  structs have reference semantics; `src/resolve.tw` depends on it heavily,
  passing `Arr` and `Dict` values into `add_deps` to be filled in.
- Does a `while` loop's condition see bindings introduced in its body on the
  next iteration? spool assumes normal scoping.
