# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Purpose

Capture and version the configuration of Erik S. Bailey's Macintosh
open-source software stack: what is installed, where, why, and how the pieces
are kept from colliding.

Read alongside this file:

- `.claude/ARCHITECTURE.md` — the host as configured now, aspect by aspect,
  and which aspects are settled.
- `.claude/DECISIONS.md` — why it is arranged that way.
- `.claude/JOURNAL.md` — what has been done here, and when.

## Repository state

Documentation only — no captured configs, no scripts, no build, test, or lint
command. **The live host is the source of truth**; everything recorded here is
a reconstruction of it. Answer questions about the setup by inspecting the
machine, then correct the docs if they disagree.

## Inspecting the host

```zsh
port installed requested          # ports explicitly asked for (the real manifest)
port outdated                     # what a selfupdate would upgrade
port select --summary             # every multi-version port and its selection
conda env list                    # miniforge environments
conda list -n esb312 --export     # pinned spec for one environment
ls /opt/dakota                    # installed Dakota releases
omz version                       # Oh My Zsh revision
xcode-select -p                   # active Xcode developer directory
xcrun --show-sdk-version          # macOS SDK the compiler targets
```

Five software sources contribute to `PATH`, so **verify which prefix a tool
resolves to before trusting it** — `command -v <tool>`. Where a tool is
supposed to come from, and why, is in `ARCHITECTURE.md`. Two cases have bitten
before and are worth checking by reflex:

```zsh
command -v pdflatex bibtex kpsewhich   # must all be /Library/TeX/texbin
command -v python                      # must be ~/miniforge3/envs/...
command -v clang                       # MacPorts wins; Apple's is `xcrun clang`
```

## Working here

- Record facts with the command that regenerates them; a number without its
  provenance goes stale silently.
- Prefer YAML for captured data.
- A change to `~/.zshrc` ordering, a prefix, or an environment default is a
  change to something marked Frozen — it needs a `DECISIONS.md` entry in the
  same change.
