# What spool needs from twill

spool is written in twill and does not run yet. This file is the reason: it is
the list of language and runtime features the source uses that `mode systems`
does not provide today, with the file that needs each one and what spool does
in the meantime.

It is meant to be read as a work queue for the language, not as a complaint.
Every entry was reached by writing real code and hitting the wall, which is the
only way this list is worth anything. Where spool has a workaround, the
workaround is described, because the ugliness of a workaround is information
about how badly the feature is wanted.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64` with bitwise operations and defined wrapping, `Str` with
length, byte indexing and slicing, `Arr[T]`, `Dict[Str, V]` with
insertion-ordered iteration, `struct`, and `read_file`. Everything spool uses
beyond that is below.

## Blocking: spool cannot work at all without these

### 1. A process interface

**Needs:** `run(program: Str, argv: Arr[Str], dir: Str) -> Str`
**Used by:** `src/vendor.tw`
**Status:** not in the systems subset at any milestone.

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
**Status:** listed in section 1.2 of the self-hosting design, not in milestone 1.

A package manager that cannot write a lockfile is a linter. This one is already
designed and only needs landing.

### 3. Directory operations

**Needs:** `list_dir(path) -> Arr[Str]`, `is_dir(path) -> Bool`,
`path_exists(path) -> Bool`, `mkdir_all(path)`, `remove_all(path)`
**Used by:** `src/vendor.tw`, `src/commands.tw`
**Status:** `list_dir` is mentioned in passing in section 1.2; the rest are not.

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
**Status:** `args`, `write_err` and `exit` are in section 1.2; `cwd` is not.

`cwd` is the one addition. spool resolves the project root by walking up from
the working directory, the way git does, and cannot start without knowing where
it is.

## Blocking: the language features the source assumes

### 5. `Str` concatenation with `+`

**Used by:** every file.
**Status:** unspecified.

`docs/self-hosting.md` gives `Bytes` a `concat` and says `Str` is convertible to
and from `Bytes` by copy, but does not say how two `Str` values are joined.
spool assumes `+` concatenates in systems mode. Every string this program builds
(the lockfile, error messages, hex digests) goes through it.

If `+` is not the answer, `str_concat(a, b)` would do, but the answer needs to
be written down; this is the single most-used operation in the source.

Related and smaller: there is no way to turn a byte value back into a one-byte
`Str`. `src/strutil.tw` works around it with a sixteen-entry `HEX` table indexed
by nibble, which is fine for hex and would not be fine for anything else. A
`str_from_byte(b: I64) -> Str` would delete that table.

### 6. The spelling of the I64 bitwise operators

**Used by:** `src/sha256.tw`, `src/strutil.tw`
**Status:** named but not specified.

Section 1.2 lists "bitwise `and or xor shl shr not` on I64" without saying
whether these are infix operators, builtin functions, or how they bind against
the arithmetic and comparison operators. spool assumes infix with
`not` prefix, and `src/sha256.tw` parenthesises every subexpression rather than
relying on a precedence that has not been decided.

`and` and `or` are also the obvious spellings for boolean conjunction, which
spool uses throughout. If they are the same operators overloaded on `Bool` and
`I64`, that should be stated; if they are different, one pair needs another
name.

### 7. `Bool` as a declared type

**Used by:** every file.
**Status:** not named in the subset.

Section 1.2 names `I64`, `Byte`, `Bytes`, `Str`, `Arr`, `Dict`, `Opt` and `Res`
as the type names, and section 1.3 makes annotation mandatory in systems mode,
but `Bool` is never listed even though comparisons obviously produce one. spool
writes `-> Bool` in a dozen places. This is probably an oversight in the
document rather than a missing feature.

### 8. `continue` in loops

**Used by:** `src/toml.tw`, `src/manifest.tw`, `src/lockfile.tw`,
`src/resolve.tw`, `src/commands.tw`
**Status:** `docs/language-guide.md` documents `return` but neither `break` nor
`continue`.

The parsers are written as a loop over lines with an early `continue` per line
shape. Without it every parser here grows a level of nesting per case, which is
exactly the readability the self-hosting document is trying to buy.

## Not blocking, but the source is worse without them

### 9. Sum types and `match`

**Would improve:** `src/toml.tw`, `src/semver.tw`, `src/manifest.tw`
**Status:** designed in section 1.2, explicitly excluded from milestone 1.

Milestone 1 says a token kind can be an `I64` constant for now and that the
ugliness of that is itself information. Here is the information, from a program
that is not a compiler.

`src/toml.tw` has a `Pair` struct with both a `value: Str` and a
`fields: Dict[Str, Str]`, plus an `is_table: Bool` saying which one is real.
That is a two-case sum written out by hand, and nothing stops a caller reading
the wrong field. `src/semver.tw` has a `Constraint` with an `I64` `kind` and a
`base` that is meaningless when the kind is "any". Both would be four lines of
`enum` and a `match` that the checker would make exhaustive.

### 10. `Res[T, E]`, `Opt[T]`, and returning two values

**Would improve:** every file.
**Status:** the `err`-field half is done (2026-08, on twill 1.7); the `"!"`-flag
half is not, and is now the ugliest thing left in the package.

**Done.** `Doc`, `Manifest`, `Lock`, `Resolution` and `Project` have lost their
`err` fields, and every function that returned a `Str` that was empty on
success returns a `Res` instead: `toml.parse`, `manifest.parse`,
`validate_name`, `validate_entry`, `lockfile.parse`, `resolve.resolve`,
`catalog_deps`, `pkghash.verify`, `commands.open`, `load_lock`,
`lock_satisfies`, `materialise`, all five `cmd_*`, and `vendor.build_catalog`
and `export`. `main.check` takes the `Res`, so there is no message to take the
length of and no way to pass one that was never read.

**Three bugs this found, none of which the tests could see**, because the suite
calls the command functions with fixture strings and never goes through `main`
or touches a file:

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

**Still to do: the `"!"`-flag convention**, which this entry rightly calls the
worse of the two. A `Str` whose first byte is a status flag -- `" "` for success
with the value following, `"!"` for failure with the message following --
survives in `toml.unquote`, `manifest.remove_dep`, `vendor.git` and
`vendor.commit_for`, with `git_ok`/`git_out` accessors existing only to make it
survivable. `Res[Str, Str]` is exactly that type and the change is mechanical.

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
**Status:** milestone 1 gives `Dict[Str, V]`; whether `V` may be a struct or an
`Arr` is not stated.

`resolve.Catalog` holds `Dict[Str, Str]` where the values are
**comma-separated version lists** and **whole rendered manifests**, re-parsed on
every read. It wants `Dict[Str, Arr[Version]]` and `Dict[Str, Manifest]`. If `V`
may already be any type, this entry is just a documentation request; if it may
not, this is the strongest argument in spool for lifting the restriction.

### 12. A sort, or a comparison-function parameter

**Would improve:** `src/strutil.tw`, `src/manifest.tw`, `src/lockfile.tw`,
`src/resolve.tw`
**Status:** no generic sort; functions are values in numeric twill, but whether
a function may be passed to a systems-mode function is not stated.

There are four near-identical insertion sorts in this source, one per element
type, differing only in the comparison. They are correct and small, and having
four of them is still four times as many places for the ordering that the
lockfile depends on to go wrong.

### 13. A `sha256` builtin

**Would improve:** `src/sha256.tw`
**Status:** not planned, and correctly so.

`src/sha256.tw` is a full SHA-256 in twill, over `I64` with explicit 32-bit
masking. It is written that way on purpose: a package manager that does not
verify what it fetched is a supply-chain hole, and a non-cryptographic hash
would detect corruption but not substitution, and substitution is the threat.

It will be slow, because it runs the compression function in the interpreter one
32-bit word at a time. For a handful of small `.tw` files at install time that
is acceptable. It would not be acceptable for large artifacts, and if spool ever
grows those, this is the first thing that should become a builtin.

It is also a good benchmark for the systems subset: it is pure `I64` and `Str`
work with no IO, so it exercises exactly the part of the subset that milestone 1
delivers, and it is verifiable against published test vectors.

### 14. Mandatory-typing policy questions the source ran into

**Status:** open questions rather than requests.

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
