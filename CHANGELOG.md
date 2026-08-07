# Changelog

## v0.1.0 (unreleased)

First cut of spool, the package manager for twill, written in twill.

It does not run. twill's `mode systems` is still being built, and spool needs a
process interface and file writing on top of it. See `docs/needs.md` for the
full list and `README.md` for the status table. Nothing below has ever executed.

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
- SHA-256 implemented in twill over `I64`, checked against the published test
  vectors including the padding boundaries at 55, 56 and 64 bytes.
- A package content hash over a length-prefixed serialisation of the file tree,
  excluding VCS metadata, and verification of it on every install.
- Vendoring into `twill_modules/`, which is where twill's `import` can reach it.
- `spool init`, `add`, `install`, `install --update`, `list`, `remove`.
- Tests for the manifest reader, versions, the resolver, the lockfile and
  SHA-256. They are written and have never been run; there is no test runner.

Deliberately not included:

- A registry. v0.1 resolves git sources only, and a dependency without a `git`
  key is a parse error that says so.
- Publishing. spool consumes packages; it does not produce them.
- Pre-release versions and build metadata. Pre-release ordering is where semver
  implementations grow their bugs and this one has not earned that complexity.
- Multiple versions of one package in a single project. twill's import brings
  names into scope, so two copies of a package would collide or diverge.
