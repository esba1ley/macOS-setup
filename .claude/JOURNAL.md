# JOURNAL.md

Work performed, newest first. Rationale belongs in `DECISIONS.md`.

## 2026-09-18 — Establish repository documentation

- Rewrote `.claude/CLAUDE.md` around the host's actual integration points
  rather than a flat list of tools: the two-layer `PATH` construction, the
  `~/.zshrc` line references where each manager enters the environment, and
  the inspection commands that regenerate every fact in it.
- Added `.claude/ARCHITECTURE.md`, `.claude/DECISIONS.md`, and this file, per
  the repository documentation convention.
- Reconstructed the host arrangement by inspection — MacPorts 2.12.6 at
  `/opt/local`, conda 25.9.1 at `~/miniforge3` with eight environments,
  two Dakota releases under `/opt/dakota`, Oh My Zsh at `master (0ee67f0)`,
  `port select` clang at `mp-clang-21`.
- Verified the 2026-08-20 TeX decision still holds: `pdflatex`, `bibtex`, and
  `kpsewhich` all resolve to `/Library/TeX/texbin`, and no MacPorts TeX ports
  are installed.
- Found two stale `/etc/paths.d` fragments — `10-pmk-global` and
  `podman-pkg` — pointing at `/pkg/env/global/bin` and `/opt/podman/bin`,
  neither of which exists.
- Repository initialized on `main`; `README.md` added, plus a `.gitignore`
  covering macOS and editor detritus.
- Made the initial commit on `main` and branched `develop` from it as the
  working trunk, initially leaving `main` untagged.
- Added Xcode as a fifth managed source and recorded the C/C++ toolchain
  split: Apple clang for compiling, MacPorts LLVM for `clang-tidy` and the
  other analysis tools. Noted that `port select` symlinks in the prepended
  `/opt/local/bin` mean a bare `clang` resolves to MacPorts 21.1.8, not Apple
  clang 21.0.0.
- Published to a public GitHub remote, `esba1ley/macOS-setup`, tagging the
  baseline commit `v0.0.0`; reviewed the content for export-controlled subject
  matter and found none.

**Next:** cut the first release on `main` once there is something substantive
to version. What gets captured as data — port manifest, conda env exports,
`~/.zshrc`, `~/.claude` — is the one aspect still Open in `ARCHITECTURE.md`,
and the one that most needs a public-visibility glance before it lands. The
stale `paths.d` fragments are removable once their origin is confirmed.
