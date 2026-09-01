# Changelog

## v0.1.0 (unreleased)

First cut of spool, the package manager for twill, written in twill.

It runs, and it fetches. `init`, `list`, `remove` and `add` do their whole job,
and `install` resolves a git dependency, clones it, vendors it into
`twill_modules/` and writes a `spool.lock` carrying the commit and the content
hash. Fetching waited on `docs/needs.md` entry 1, a process interface in twill,
which now exists as `run(program, argv, dir) -> Res[Str, Str]` -- the signature
that entry asked for. All fourteen entries in that file are delivered.
`README.md` has the status table.

**This needs a twill newer than 1.7.1.** `run` is not in a twill release yet, so
until one exists this code checks and runs only against a twill built from the
language repository, and CI — which pins `v1.7.1` on purpose, so that a green
run means a known compiler — fails at `twill check` with `unknown name "run"`.
That is the correct failure rather than something to work around: the pin moves
when the release exists, and the `^1.7.0` in `spool.toml` moves with it.

Two things changed here to meet it:

- `git()` in `src/vendor.tw` is one line. It used to unwrap a status byte the
  old `run` put in front of its output, because the language could not return
  two values; `docs/needs.md` entry 10 called that the ugliest thing in spool
  and this was its last hiding place.
- `published_versions` answers a `Res` rather than an `Arr`. A repository that
  clones and carries no readable tag publishes no versions; one that cannot be
  reached at all -- no `git` on PATH, `TWILL_NO_EXEC` set, a URL nobody can
  clone -- is a different failure, and folding both into an empty list made
  spool blame the repository for the local problem.

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
