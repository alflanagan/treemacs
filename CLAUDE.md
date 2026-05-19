# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working
with code in this repository.

## Commands

Dependencies are managed with [Cask](https://github.com/cask/cask). All
`make` targets call `cask` to install/update `.cask/` first.

```bash
# Install/update dependencies (also run automatically by make targets)
cask install

# Byte-compile all source files
make compile

# Run the test suite (buttercup)
make test

# Full lint pass: byte-compile to /dumpy + checkdoc
make lint

# Checkdoc only
make checkdoc

# Remove compiled .elc files
make clean

# Launch a clean Emacs with dreams loaded (for manual testing)
make clean-start
```

To run a single buttercup spec file directly:

```bash
cask exec buttercup -L src/elisp -L src/extra -L . \
  --eval "(defvar treemacs-no-load-time-warnings t)" \
  test/treemacs-test.el
```

Byte compilation uses `byte-compile-error-on-warn t`, so all warnings are
treated as errors.

## Architecture

### Directory layout

- `src/elisp/` — Core package source files (all loaded as part of the
  main `treemacs` feature)
- `src/extra/` — Optional integration packages: `treemacs-evil`,
  `treemacs-magit`, `treemacs-projectile`, `treemacs-poops`,
  `treemacs-perspective`, `treemacs-tab-bar`, `treemacs-icons-dired`,
  `treemacs-all-the-icons`, `treemacs-mu4e`. Each is independently
  `require`-able; none are loaded by default.
- `src/scripts/` — Python 3 helper scripts invoked asynchronously for
  git status, ignored-file detection, collapsed-directory analysis, git
  commit diffs, and mail counting.
- `test/` — Single `buttercup`-based test file plus `checkdock.el` (a
  thin wrapper that runs `checkdoc` across all source files).

### Core data model

**DOM (`treemacs-dom.el`)** — Each treemacs buffer holds a buffer-local
hash-table `treemacs-dom` mapping node keys (typically absolute file
paths, represented as lists for nested nodes) to `treemacs-dom-node`
structs. Nodes store their parent, children, buffer position marker,
refresh flag, and collapse keys. This is the authoritative in-memory
model of the visible tree; rendering reads and writes it.

**Workspaces & projects (`treemacs-workspaces.el`)** — A
`treemacs-workspace` is a named list of `treemacs-project` structs.
Each project carries a name, a root path, and a path-status (local,
disconnected, extension, etc.). The global list `treemacs--workspaces`
holds all workspaces; `treemacs--disabled-workspaces` holds hidden
ones. Workspaces are persisted to an org-mode-compatible file by
`treemacs-persistence.el`.

**Scope (`treemacs-scope.el`)** — Associates each "scope" (by default,
an Emacs frame) with a `treemacs-scope-shelf` that holds the treemacs
buffer and current workspace for that scope. Scope types are eieio
classes registered in `treemacs-scope-types`; the current type is
stored in `treemacs--current-scope-type`. The extra packages
`treemacs-persp`, `treemacs-perspective`, and `treemacs-tab-bar` each
register their own scope type.

### Rendering pipeline

`treemacs-rendering.el` is explicitly marked performance-critical and
compiled with `(cl-declaim (optimize (speed 3) (safety 0)))`. It writes
text with text properties into the (temporarily writable) treemacs
buffer and keeps the DOM in sync. Tags are handled separately in
`treemacs-tags.el`, which queries imenu indices.

### Async subsystem (`treemacs-async.el`)

Long-running operations (git status, single-file git status, ignored
file detection, collapsed-directory scanning, git commit diffs, mail
counts) are farmed out to the Python scripts in `src/scripts/` via
`pfuture` (async subprocesses with callbacks). Results are merged back
into the DOM/display on completion.

### Extension APIs

There are two extension APIs:

- **`treemacs-extensions.el`** (older) — Lets external packages inject
  custom rendered nodes at the top or bottom of any project via
  `treemacs-define-{project,workspace,...}-extension`.
- **`treemacs-treelib.el`** (newer, v1.1) — A higher-level API using
  `treemacs-extension` structs with declared children/key/label/icon
  slots. Supports async loading. Prefer this API for new extensions.

### Naming conventions

- Public symbols: `treemacs-*`
- Private/internal symbols: `treemacs--*` (double dash)
- Struct field accessors: `treemacs-<type>-><field>`
  (e.g. `treemacs-project->name`)
- Struct constructors: `treemacs-<type>->create!`
- The `treemacs-import-functions-from` macro (defined in
  `treemacs-macros.el`) generates `declare-function` stubs to allow
  cross-module calls without hard `require` cycles.

### Load-order sensitivity

`treemacs-macros.el` must be available before any file that uses its
macros. It is `require`d under `eval-when-compile` in most modules.
`treemacs.el` is the top-level entry point that pulls in all core
modules in dependency order. `treemacs-peek-mode.el` is lazy-loaded
and does not appear in the compile list.
