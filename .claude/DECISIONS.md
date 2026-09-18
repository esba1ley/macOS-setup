# DECISIONS.md

Why the machine and this repository are arranged as they are. Newest first.
Entries dated before this file existed were reconstructed from the live
configuration and are marked as such.

## 2026-09-18 — Tag the baseline `v0.0.0` and publish publicly
**Status:** Accepted
**Context:** GitFlow requires every commit on `main` to carry a semantic
version tag, but the initial commit is documentation with no captured
configuration in it — nothing that deserves to be called a feature release.
Deferring the tag (the entry below) left `main` knowingly out of compliance.
**Decision:** Tag the baseline commit `v0.0.0`, which satisfies the tagging
rule without claiming a release, and publish `main` and `develop` to a public
GitHub remote at `esba1ley/macOS-setup`.
**Consequences:** `main` is compliant from its first commit, and releases
advance from `0.0.0` under `bump2version` once there is captured configuration
to version. Public visibility makes the repository a readable inventory of
this machine — local paths, installed software and versions, environment
names — so each commit needs that glance before it goes out; the 2026-09-18
review found no export-controlled subject matter. Rejected: `v0.1.0`, which
would name a release containing no captured configuration; and a private
remote, which the public one forecloses without a history rewrite.

## 2026-09-18 — Compile with Apple clang, analyze with MacPorts LLVM
**Status:** Accepted (recorded retroactively)
**Context:** Apple's clang is the build the platform expects — integrated with
the macOS SDK, matching the system headers and linker — but Apple strips the
LLVM extras, shipping no `clang-tidy`, `clang-format`, or `clang-doc`. The C
and C++ conventions require `clang-tidy`.
**Decision:** Install both. Full Xcode supplies the compiler and SDK; MacPorts
supplies LLVM at the same major version (`mp-clang-21` against Apple clang 21)
for the analysis tools.
**Consequences:** The two must be held at the same major version by hand —
`port select --set clang mp-clang-<N>` after an Xcode upgrade — or `clang-tidy`
and the compiler disagree about standard-library headers. Because `port
select` writes its symlinks into the prepended `/opt/local/bin`, a bare `clang`
is the MacPorts one; Apple's must be named explicitly. Rejected: MacPorts LLVM
as the compiler too, which drifts from the SDK Apple ships; and doing without
`clang-tidy`, which the coding conventions require.

## 2026-09-18 — Adopt GitFlow branches, defer semantic versioning
**Status:** Superseded by 2026-09-18 — Tag the baseline `v0.0.0` and publish
publicly
**Context:** The standing convention is GitFlow — `main` carrying only tagged
releases, `develop` as the trunk — and every commit on `main` carrying a
semantic version tag. The first commit here is documentation, with nothing yet
that deserves to be called a release.
**Decision:** Create both branches now. Leave the initial commit on `main`
untagged and add no `bump2version` configuration until there is a release to
cut.
**Consequences:** The branch structure is in place before it is needed, so no
history rewrite later. Until the first tag lands, `main` is knowingly out of
compliance with the tagged-release rule; tag it once the captured-configuration
question in `ARCHITECTURE.md` is settled and there is something substantive to
version. Rejected: tagging `v0.1.0` immediately, which would name a release
containing no captured configuration; and a single `main` branch, which would
have to be split later.

## 2026-09-18 — Defer choosing between documenting the host and rebuilding it
**Status:** Deferred
**Context:** A machine-configuration repository can be a written record of a
hand-tended host, or a declarative definition that can rebuild one (Nix, an
Ansible playbook, a Brewfile-style manifest). The two imply very different
contents, and committing to the wrong one wastes the capture effort.
**Decision:** Deferred. For now the repository documents; nothing here claims
to be executable or authoritative over the host.
**Consequences:** Documentation can drift from the machine with nothing to
detect it, so every recorded fact carries the command that regenerates it.
Choosing the declarative path later means rewriting captured artifacts, not
merely adding to them. Revisit once it is clear whether the goal is disaster
recovery or a second machine.

## 2026-09-18 — Record the existing host arrangement as Frozen
**Status:** Accepted
**Context:** The prefix isolation, Python layout, and Dakota scheme below
predate any written record and were reconstructed by inspecting the host. They
could be logged as provisional, or accepted as settled.
**Decision:** Treat them as Frozen in `ARCHITECTURE.md`. They have been stable
and working for over a year, and changing one now requires a decision entry.
**Consequences:** Raises the bar for casual changes to `PATH` ordering or
environment layout, which is the point. The reconstruction is only as good as
the inspection — anything the live host contradicts should be corrected here
rather than argued with.

## 2026-09-18 — Keep each package manager in its own prefix
**Status:** Accepted (recorded retroactively; arrangement predates this log)
**Context:** MacPorts, miniforge, MacTeX, Dakota, and Apple's own system all
ship overlapping software — Python above all. Left to themselves they shadow
one another by `PATH` order, and the winner varies by shell and by login type.
**Decision:** One prefix per source, no cross-installation, precedence set
explicitly by `PATH` ordering in `~/.zshrc`. MacPorts is the system package
manager; miniforge owns every Python environment; Apple's `/usr/bin/python3`
is never installed into.
**Consequences:** Some software is installed twice at different versions, and
`command -v` is required before trusting any tool. In exchange, removing one
manager cannot break another. Rejected: a single manager for everything, which
no one of them covers; and MacPorts' Python ports for project work, which
conflicts with conda environment management.

## 2026-09-18 — Select Dakota by variable, not by replacement
**Status:** Accepted (recorded retroactively; in place since July 2025)
**Context:** Dakota ships as a self-contained versioned tree. Upgrading could
overwrite in place, or keep releases side by side.
**Decision:** Unpack releases side by side under `/opt/dakota` and select one
with `DAKOTA_RELEASE` in `~/.zshrc`, deriving `PATH` and `PYTHONPATH` from it.
**Consequences:** Switching releases, or falling back after a bad upgrade, is
a one-line edit; disk cost is one full tree per release. Rejected: a `current`
symlink, which hides the active version from anyone reading `~/.zshrc`.

## 2026-09-18 — Activate esb312 in every interactive shell
**Status:** Accepted (recorded retroactively)
**Context:** conda leaves `base` active by default, which invites installing
project packages into it.
**Decision:** Activate `esb312` at the end of `~/.zshrc`, matching the
preferred Python version.
**Consequences:** A bare `python` is always a real working environment, and
`base` stays clean. The cost is that the active environment is implicit —
scripts and cron jobs that do not source `~/.zshrc` get a different Python,
and the default must be edited here when the preferred version advances.

## 2026-08-20 — Make MacTeX the sole TeX installation
**Status:** Accepted
**Context:** MacPorts TeX ports and MacTeX were both installed. A build
resolved `pdflatex` from MacTeX but `bibtex` and `kpsewhich` from MacPorts;
BibTeX searched a different `texmf` tree, failed to find `plainnat.bst`, and
wrote an empty `.bbl`. `latexmk` then aborted before the label-resolving
passes, leaving a stale PDF in which every citation and reference rendered as
`??` — a build that appeared to succeed and had not.
**Decision:** Remove the MacPorts TeX ports. MacTeX at `/Library/TeX/texbin`
is the only TeX tree; CTAN packages are installed with `tlmgr`, never `port`.
**Consequences:** A mixed toolchain can no longer fail silently; the check is
that `pdflatex`, `bibtex`, and `kpsewhich` all resolve to one directory
(verified 2026-09-18). Build scripts pin the toolchain rather than trusting
`PATH`. Poppler utilities (`pdfinfo`, `pdftotext`) are not part of MacTeX and
must not be treated as build dependencies. Full diagnosis and the Makefile
snippet live in `~/.claude/rules/environment.md`.
