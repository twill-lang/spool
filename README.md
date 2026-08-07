<p align="center">
  <a href="https://github.com/martin-k-m/twill">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/martin-k-m/twill/main/assets/twill-mark-glow.png">
      <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/martin-k-m/twill/main/assets/twill-mark.png">
      <img alt="twill" src="https://raw.githubusercontent.com/martin-k-m/twill/main/assets/twill-mark.png" width="72">
    </picture>
  </a>
</p>

<h1 align="center">spool</h1>

<p align="center">
  <b>The package manager for <a href="https://github.com/martin-k-m/twill">twill</a>.</b><br>
  Written in twill, against a language subset that does not exist yet.
</p>

<p align="center">
  <a href="https://github.com/martin-k-m/spool/actions/workflows/ci.yml"><img alt="CI" src="https://img.shields.io/github/actions/workflow/status/martin-k-m/spool/ci.yml?branch=main&style=flat-square&label=source%20gate&labelColor=33231A&color=E3A76F"></a>
  <img alt="version 0.1.0" src="https://img.shields.io/badge/version-0.1.0-E3A76F?style=flat-square&labelColor=33231A">
  <img alt="status: does not run" src="https://img.shields.io/badge/status-does%20not%20run-F2DCC6?style=flat-square&labelColor=33231A">
  <a href="https://github.com/martin-k-m/twill"><img alt="written in twill" src="https://img.shields.io/badge/written%20in-twill-7FE3C4?style=flat-square&labelColor=12332C"></a>
  <img alt="dependencies: none" src="https://img.shields.io/badge/dependencies-none-F2DCC6?style=flat-square&labelColor=33231A">
  <a href="LICENSE"><img alt="MIT" src="https://img.shields.io/badge/license-MIT-E3A76F?style=flat-square&labelColor=33231A"></a>
</p>

---

## spool does not run yet

Read that first, because everything below describes behaviour spool is *written*
to have, not behaviour it has.

spool is written in twill, in `.tw` files, and it uses `mode systems`, the
systems subset of the language described in
[`docs/self-hosting.md`](https://github.com/martin-k-m/twill/blob/main/docs/self-hosting.md)
in the twill repository. That subset is being built. Until it lands, and until
twill gains a process interface and the ability to write files, none of the
commands below execute. There is no binary to download and `spool install` will
not install anything.

This repository is the program, written out in full, ahead of the language that
runs it. That is deliberate. Writing a real program against a subset is how you
find out what the subset is missing, and what it is missing is written down in
[`docs/needs.md`](docs/needs.md): one entry per feature, naming the file that
needs it and what spool does instead in the meantime.

**That list is the useful output of this repository today.** The package manager
is what the list was produced by.

## Contents

- [Status](#status)
- [What spool is meant to do](#what-spool-is-meant-to-do)
- [The manifest](#the-manifest)
- [The lockfile](#the-lockfile)
- [How a twill project consumes a spool package](#how-a-twill-project-consumes-a-spool-package)
- [Repository layout](#repository-layout)
- [Dependencies](#dependencies)
- [Contributing](#contributing)
- [License](#license)

## Status

"Written, unrun" means the twill source exists, is complete, and has tests that
have never executed. Treat it as a design that compiles in someone's head.

| Piece | State |
| --- | --- |
| `spool.toml` reader, a hand-rolled TOML subset | written, unrun |
| Version parsing and `^` constraints | written, unrun |
| Dependency resolver, pure and network-free | written, unrun |
| `spool.lock` writer and reader, deterministic | written, unrun |
| SHA-256, in twill, verified against published vectors | written, unrun |
| Package content hashing and verification | written, unrun |
| Vendoring into `twill_modules/` | written, blocked on file and process IO |
| `add` / `install` / `list` / `remove` | written, blocked on file and process IO |
| Tests | written, blocked on a test runner |
| A registry | not planned for v0.1. Git sources only |
| Publishing packages | not in scope. spool consumes, it does not publish |
| Anything running end to end | no |

CI reflects that honestly. There is no build step and no test run, because
neither is possible; the workflow gates the things that *are* checkable today,
including that this README still says spool does not run. When `mode systems`
lands, the gate becomes `twill run tests/*.tw` and every existing check stays.

## What spool is meant to do

A package manager for a language whose module system is `import "path"` resolving
to a file on disk. It is built to fit that and not to imitate cargo.

Five things, and it stops there.

1. **A manifest.** `spool.toml` declares a package name, a version, and
   dependencies with a version constraint and a git source.
2. **Four commands.** `add`, `install`, `list`, `remove`, plus `init`.
3. **A lockfile.** `spool.lock` records the exact commit and content hash of
   every dependency. You commit it. Deterministic output, sorted, stable and
   diffable, matters more here than anything else in the tool.
4. **Vendoring.** Packages land in `twill_modules/` inside your project, because
   that is the only place twill's `import` can reach them.
5. **Integrity.** Every package is hashed on install and verified on every
   install after that. A package manager that does not check what it fetched is
   a supply-chain hole, and twill's whole proposition is a zero-dependency
   toolchain.

### v0.1 is git-source only

There is no registry, there is no index, and there is no `spool publish`. A
dependency names a git URL or it is rejected at parse time with an error saying
so. If a registry ever exists it will be a field in the manifest, not a reshaping
of it.

## The manifest

```toml
name = "myproject"
version = "0.3.1"
entry = "src/main.tw"

[dependencies]
tensorstats = { version = "^1.2.0", git = "https://github.com/example/tensorstats" }
plotting    = { version = "0.4.0",  git = "https://github.com/example/plotting" }
```

`version` on a dependency is either an exact version (`"1.2.0"`), a caret range
(`"^1.2.0"`), or `"*"`. Tilde ranges, comparison operators and multi-part ranges
are **rejected**, not ignored. Quietly widening a constraint someone wrote
narrowly is a worse failure than refusing to read it.

`entry` is documentation. twill has no notion of a package entry point, so spool
records it and shows it in `spool list`; it does not enforce it.

`spool.toml` is read by a hand-rolled parser covering comments, one-level
`[table]` headers, quoted strings, and single-line inline tables. That is the
whole subset. It is not TOML. It is a file that both a TOML parser and this
reader will accept, and only inside that subset do the two agree. Escape
sequences are not interpreted, so a Windows-style path reads back byte for byte.

## The lockfile

```toml
# spool.lock -- generated by spool. Commit this file.

version = 1

[[package]]
name = "tensorstats"
version = "1.2.4"
git = "https://github.com/example/tensorstats"
commit = "9f1c0e6a3b2d4f5081726354a9b0cde1f2345678"
hash = "sha256:1b2c3d4e5f60718293a4b5c6d7e8f90a1b2c3d4e5f60718293a4b5c6d7e8f90a"
```

Commit it. `spool install` with a lockfile that still satisfies the manifest does
no version resolution and talks to no network: it uses the recorded commits
exactly. `spool install --update` is what moves versions.

Two fields, two different questions:

| Field | Answers | Catches |
| --- | --- | --- |
| `commit` | what did the tag point at? | a tag that moved under you |
| `hash` | what actually arrived on disk? | a mirror serving different bytes under that same commit |

spool records both, because neither one answers for the other.

## How a twill project consumes a spool package

This part is specific to twill, so it is spelled out end to end.

twill resolves a non-`std/` import as a **file path**, relative to the importing
file first and the working directory second. There is no module search path and
no global package store to point it at. So spool vendors into the project:

```
myproject/
  spool.toml
  spool.lock
  twill_modules/          <- written by spool, gitignored
    tensorstats/
      spool.toml
      tensorstats.tw
  src/
    main.tw
```

**1. Declare and install.**

```
spool add tensorstats https://github.com/example/tensorstats
```

**2. Gitignore the vendor directory, commit the two spool files.**

```
twill_modules/
```

`spool.toml` and `spool.lock` are what reproduce the tree. The tree itself is not
source.

**3. Import it, by path, from your project root.**

```rust
import "twill_modules/tensorstats/tensorstats.tw" as stats

let m = stats.mean(xs)
```

Or unaliased, which drops the package's definitions into your scope:

```rust
import "twill_modules/tensorstats/tensorstats.tw"
```

**4. Run from the project root**, because that is what makes the path resolve:

```
twill run src/main.tw
```

The import path is relative to the working directory, so `twill run` from a
subdirectory will not find it. That is twill's rule, not spool's, and the honest
statement is that the import line is verbose and the working-directory dependency
is a sharp edge. Both would go away if twill grew a package-aware import form.
Neither is something spool can fix from outside the language.

## Repository layout

```
main.tw              argument parsing, help, exit codes
src/strutil.tw       byte and string helpers; everything else builds on these
src/toml.tw          the spool.toml reader
src/semver.tw        versions and `^` constraints
src/manifest.tw      spool.toml model, parse, render, add/remove a dependency
src/resolve.tw       the resolver: pure, no IO, testable from a literal table
src/sha256.tw        SHA-256 over I64
src/pkghash.tw       the package content hash and its verification
src/lockfile.tw      spool.lock render and parse
src/vendor.tw        git, the filesystem, and nothing else in spool touches them
src/commands.tw      add, install, list, remove, init
tests/               tests, named as sentences
docs/needs.md        what the language still has to provide
```

The resolver being a pure function of a manifest and a catalog is the single most
important structural decision here. Every resolution test is a literal table of
versions with no network and no filesystem, and `src/vendor.tw` is the only file
that reaches outside the process.

## Dependencies

None. Not "few". None. No third-party twill packages, no Go, no shell scripts
doing the real work, no vendored anything. SHA-256 is implemented in this
repository in twill for exactly that reason, and it is slower than a builtin
would be. That cost is written down in `src/sha256.tw` rather than hidden.

## Contributing

The most useful contribution right now is not code. It is a correction to
[`docs/needs.md`](docs/needs.md): a feature listed there that the language already
has, a workaround that is worse than described, or a missing entry found by
reading the source.

Prose here follows twill's house style. No em dashes, and CI checks for them.

## License

MIT. See [LICENSE](LICENSE).
