# Changelog

## v0.1.0 (unreleased)

First cut of spool, the package manager for twill, written in twill.

It runs, on twill 1.7.1. `init`, `list` and `remove` do their whole job;
`install` writes the lockfile and the vendor directory; `add` writes the
dependency into `spool.toml` and then stops where it would fetch, because twill
has no process interface and spool fetches by running `git`. That is the one
thing still missing, and `docs/needs.md` entry 1 is the whole of it. `README.md`
has the status table.

Added:

- `spool.toml`, read by a hand-rolled parser over a documented TOML subset:
  comments, one-level tables, quoted strings, single-line inline tables. Escape
  sequences are not interpreted.
- Version parsing and two constraint forms, exact and `^`. Tilde ranges,
  comparison operators and multi-part ranges are rejected rather than ignored.
- A dependency resolver that is a pure function of a manifest and a catalog:
  one version per package name, highest that satisfies every constraint,
  deterministic, with errors that name the dependency responsible.
- `spool.lock` with the exact commit and content hash of every package, sorted
  by name and rendered in a fixed field order so the same resolution always
  produces the same bytes.
- Package hashing over twill's `std/hash`, checked against the published
  SHA-256 test vectors including the padding boundaries at 55, 56 and 64 bytes.
  This began as `src/sha256.tw`, a SHA-256 written here in twill; it moved into
  the standard library so the toolchain has one implementation of a digest that
  everything must agree on byte for byte.
- A package content hash over a length-prefixed serialisation of the file tree,
  excluding VCS metadata, and verification of it on every install.
- Vendoring into `twill_modules/`, which is where twill's `import` can reach it.
  Written, and blocked on a process interface for git.
- `spool init`, `add`, `install`, `install --update`, `list`, `remove`.
- Six test suites, for the manifest reader, versions, the resolver, the
  lockfile, hashing and the status lines. `twill test tests` runs them: 6
  file(s), 6 passed, 0 failed.

Deliberately not included:

- A registry. v0.1 resolves git sources only, and a dependency without a `git`
  key is a parse error that says so.
- Publishing. spool consumes packages; it does not produce them.
- Pre-release versions and build metadata. Pre-release ordering is where semver
  implementations grow their bugs and this one has not earned that complexity.
- Multiple versions of one package in a single project. twill's import brings
  names into scope, so two copies of a package would collide or diverge.
