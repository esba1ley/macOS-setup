# ARCHITECTURE.md

What this repository is, and what the host it describes looks like now.
Rationale lives in `DECISIONS.md`; work performed lives in `JOURNAL.md`.

## Repository Contents
**Status:** Open

The repository holds documentation only — `README.md`, `CLAUDE.md`, and these
three living documents. No configuration has been captured as data yet, so the
live host remains the source of truth for every fact recorded here.

Unresolved: which artifacts are worth committing (port manifest, conda env
exports, `~/.zshrc`, `~/.claude`), and whether they are captured as snapshots
or as inputs that could rebuild the machine.

## PATH Construction
**Status:** Frozen

`PATH` is built in two layers, and both matter when diagnosing which copy of a
tool wins:

1. **System layer** — `/etc/paths` then `/etc/paths.d/*`, assembled by
   `path_helper` from `/etc/zprofile`. Supplies Apple's `/usr/bin`, MacTeX
   (`/Library/TeX/texbin`), and XQuartz (`/opt/X11/bin`).
2. **User layer** — `~/.zshrc`, which *prepends* and therefore takes
   precedence: MacPorts, Dakota, miniforge, `~/.local/bin`.

`~/.zprofile` is owned by Docker Desktop and adds `~/.docker/bin`; it is not a
place to put anything by hand.

Known cruft: `/etc/paths.d/10-pmk-global` and `/etc/paths.d/podman-pkg` point
at `/pkg/env/global/bin` and `/opt/podman/bin`, neither of which exists.

## Package Manager Isolation
**Status:** Frozen

Four software sources coexist, each in its own prefix, none permitted to
shadow another:

| Source | Prefix | Role |
|--------|--------|------|
| MacPorts | `/opt/local` | System libraries, compilers, CLI tools |
| miniforge3 | `~/miniforge3` | All Python environments |
| MacTeX | `/Library/TeX` | The entire TeX toolchain |
| Dakota | `/opt/dakota` | Self-contained, vendor-shipped |

Apple's `/usr/bin/python3` is left untouched and is never installed into.
`~/.local/bin` holds user-level binaries outside any manager — currently
`claude` and `sdfast`.

## Python Environments
**Status:** Frozen

miniforge owns Python. Environments are named `esb<major><minor>` for general
use (`esb311`, `esb312`, `esb313`) and by purpose otherwise (`cv313`,
`dev313`, `matchstick`, `xmas_lights`). `~/.zshrc` activates `esb312` at the
end of every interactive startup, so a bare `python` is 3.12 — never `base`,
never Apple's.

## Dakota Installation
**Status:** Frozen

Releases are unpacked side by side under `/opt/dakota` and one is selected by
the `DAKOTA_RELEASE` variable in `~/.zshrc`; `PATH` and `PYTHONPATH` derive
from it, so switching releases is a one-line edit. Present:
`dakota-6.22.0-...-gui_cli` (selected) and `dakota-6.20-...-cli`.

Dakota injects its own Python package tree onto `PYTHONPATH`
(`share/dakota/Python`, providing `dakota` and `muq`). This crosses the
otherwise clean miniforge boundary and is the one deliberate exception to the
isolation above.

## TeX Toolchain
**Status:** Frozen

MacTeX is the sole TeX installation; no MacPorts TeX ports are installed.
`pdflatex`, `bibtex`, and `kpsewhich` all resolve to `/Library/TeX/texbin`,
which is the invariant that matters — see `DECISIONS.md` (2026-08-20) for what
breaks when they do not.

## Version Control
**Status:** Open

Git repository initialized 2026-09-18 on `main`, no commits yet, no remote,
and no commit signing or DCO sign-off configured.

Unresolved: whether the repository gains a remote and whether that remote is
public — the content is personal machine configuration, but it does enumerate
installed software, local paths, and project names; and whether a docs-only
repository warrants the full GitFlow trunk structure.
