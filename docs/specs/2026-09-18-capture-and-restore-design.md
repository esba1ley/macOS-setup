# Capture & Restore — Design

**Date:** 2026-09-18
**Status:** Approved, not yet implemented
**Sub-project:** A of three (B: release monitoring, C: shareable write-up)

## Purpose

Let this repository recreate the machine it describes, and keep doing so as the
machine changes. Today the repository documents the host; a reader can learn how
it is arranged but cannot rebuild it. This design closes that gap for everything
that can honestly be automated, and makes the remainder an explicit checklist
rather than an implied one.

Independence from Time Machine is a requirement, not a nicety: the restore path
must work when no incremental backup is available.

## Scope

**In:** MacPorts ports, conda environments, VS Code extensions, the six
toolchain applications no package manager owns, three git-cloned
configuration directories, and an allowlisted set of dotfiles.

**Out:** Personal configuration, which lives in the private companion
repository `esba1ley/claude-user-settings` and is cloned rather than copied.
Also application data, documents, secrets of any kind, open-source
applications outside the build toolchain (Blender, FreeCAD, GIMP, Inkscape,
OBS, Firefox, Raspberry Pi Imager), and all commercial software. These get a
single line in `RESTORE.md` noting they are installed separately; no versions
are tracked.

**Deferred to B and C:** the weekly release report and the public-facing
narrative. This design must not grow features to serve them, but its manifests
are B's input and should stay machine-readable for that reason.

## Decisions

Four choices were settled during design. Each records its rejected alternative
because the reasoning matters more than the outcome.

1. **Guided per-layer scripts**, not a one-command bootstrap. macOS gates the
   first stretch of any fresh install — Xcode licensing, admin authentication,
   App Store sign-in — so an unattended run is a promise the platform breaks.
   Per-layer scripts are also safe to run on a *working* machine, which is what
   lets them double as drift-sync. Rejected: a single `bootstrap.sh`, whose
   value materialises only on a day that may never come, and which leaves
   unclear state when it fails midway.

2. **Allowlist capture; secrets are never captured at all.** The repository is
   public. Encrypting secrets in place, or splitting them into a private repo,
   both create a bootstrap paradox: restoring a fresh Mac would need a
   decryption key, or SSH access, that is itself the thing being restored — and
   Time Machine, the usual escape hatch, is explicitly out. Anything genuinely
   secret is restored by hand from a password manager and listed in
   `RESTORE.md`. Rejected: `age`/`git-crypt` encryption, and a private vault
   repository.

   This does **not** conflict with the private companion repository in decision
   5. What was rejected is a private repo holding *secrets*, which is circular:
   reaching it needs the SSH key it contains. A private repo holding personal
   but non-secret configuration is reached *after* keys are restored by hand,
   so the chain has a manual root rather than a cycle.

3. **Copy-in, not symlinks.** `bin/capture-dotfiles` copies allowlisted files
   into the repository; you review the diff before committing. This puts an
   explicit gate between a live config and a world-readable repository, which
   matters because the failure mode is permanent. It is also safe with
   applications that rewrite files wholesale rather than editing in place —
   `p10k configure` regenerates `.p10k.zsh` entirely. Rejected: a stow-style
   symlink farm, and a bare repository with `$HOME` as work tree.
   The cost is drift between capture runs; sub-project B's weekly trigger
   cancels it by reporting drift, which is why `bin/check-drift` exists here.

4. **Toolchain-only vendor scope.** The six applications tracked are precisely
   those that gate something else. Rejected: tracking every non-Apple
   application, which would publish a full inventory including tax software to
   a public repository and dilute the open-source story; and a narrower
   toolchain list that omitted VS Code.

5. **Personal configuration lives in a private companion repository**, not in
   this one. `~/.claude` is already a clone of
   `git@github.com:esba1ley/claude-user-settings.git` (private, branch
   `develop`), carrying `CLAUDE.md` and `rules/`. Those files hold name,
   employer, job title, education and personal inventory — no credential
   pattern in them, so a secret scanner passes them straight through. The
   protection against publishing personal data is **repository separation, not
   scanning**; a scanner cannot detect "this is personal." Rejected: copying
   them into this public repository behind the secret scan, which is what an
   earlier draft of this design did.

## Architecture: five ownership classes

The inventory is split by *who owns the install*, because that determines
whether restore can be scripted at all.

| Class | Members | Capture | Restore |
|---|---|---|---|
| Package-managed | 18 requested ports | `port installed requested` | `port install` |
| Environment-managed | 8 conda environments | `conda env export --from-history` | `conda env create` |
| App-managed | 62 VS Code extensions | `code --list-extensions` | install loop |
| Vendor-installed | 6 toolchain applications | version + source only | documented checklist |
| Repo-managed | 3 git clones | URL, branch, destination | `git clone` |

Two of these capture commands share a principle worth stating plainly:
`installed requested` and `--from-history` both record **what was asked for**,
never the solved dependency closure. 18 ports rather than 216; the packages you
named rather than everything conda resolved. This keeps manifests readable,
portable across architectures, and stable under upstream dependency churn.

### The vendor class gates everything else

Vendor applications are not a footnote. They are the prerequisite layer:

```
restore-dotfiles   no prerequisite (pure file copy)
restore-ports      needs MacPorts, which needs Xcode + Command Line Tools
restore-conda      needs miniforge
restore-vscode     needs Visual Studio Code.app
```

MacTeX, Dakota, and Docker Desktop gate documented workflows rather than a
script, but are tracked on the same terms. Every `restore-*` script therefore
opens by checking its prerequisite and failing fast with a pointer to the
`RESTORE.md` step that satisfies it. A restore script that runs without its
prerequisite and half-succeeds is worse than one that refuses.

Docker Desktop and VS Code additionally **self-update** through their own
updaters, so their versions move without `port outdated` ever mentioning it.
That property is sub-project B's problem, but `vendors.yml` is where B will
look, so it is captured here.

### Repo-managed configuration

Three directories are not files to copy but git clones to reproduce:

| Destination | Origin | Access |
|---|---|---|
| `~/.oh-my-zsh` | `ohmyzsh/ohmyzsh` | public, HTTPS |
| `~/.oh-my-zsh/custom/themes/powerlevel10k` | `romkatv/powerlevel10k` | public, HTTPS |
| `~/.claude` | `esba1ley/claude-user-settings` | **private, SSH** |

Capture for this class is near-trivial — record URL, branch, destination — and
restore is a clone. The content is already versioned in its own repository, so
duplicating it here would create two sources of truth for the same files.

Two practical constraints:

- The private clone needs SSH keys, which are a manual restore step. This is
  why manual prerequisites move earlier in the restore order than they would
  otherwise sit.
- `~/.claude` may already exist and be non-empty when restore runs, because
  Claude Code creates it on first launch. `git clone` refuses a non-empty
  destination, so `restore-repos` uses `git init` + `remote add` + `fetch` +
  `checkout -f <branch>` instead, which is also the idempotent form: re-running
  it against an existing clone is a fetch and a no-op checkout.

## Repository layout

```
bin/
  capture              runs every capture-* in order
  capture-ports        capture-conda      capture-vscode
  capture-vendors      capture-dotfiles   capture-repos
  restore-ports        restore-conda      restore-vscode
  restore-dotfiles     restore-repos
  check-drift          capture to temp, diff against repo, report
manifests/
  ports.txt
  conda/<env>.yml
  vscode-extensions.txt
  vendors.yml
  repos.yml
dotfiles/
  allowlist.yml
  zshrc  p10k.zsh  condarc  gitconfig  emacs
docs/
  RESTORE.md           running order, Apple-gated steps, manual items
  CAPTURE.md           how to add something to the allowlist
  specs/               this document
test/
  *.bats
```

There is no `dotfiles/claude/`. `~/.claude` is repo-managed (see above) and
never copied here. This repository's own `.claude/` holds `ARCHITECTURE.md`,
`DECISIONS.md`, `JOURNAL.md`, and `CLAUDE.md`, and is authored content rather
than captured state.

## Data formats

All manifests are YAML or plain text, per the repository convention, and each
carries the command that regenerates it as a leading comment.

`manifests/ports.txt` — one port name per line, version-free. Versions belong to
the MacPorts tree at restore time; pinning them here would guarantee staleness
and defeat `port outdated`.

`manifests/conda/<env>.yml` — one file per environment, `--from-history`.

`manifests/vscode-extensions.txt` — one `publisher.extension` identifier per
line.

`manifests/vendors.yml`:

```yaml
- name: Xcode
  version: "27.0"          # build 27A266a
  source: "Mac App Store"
  check: "xcode-select -p"
  gates: [macports, command-line-tools]
  self_updates: false
- name: Docker Desktop
  version: "4.90.0"
  source: "https://www.docker.com/products/docker-desktop/"
  check: "docker version --format '{{.Server.Version}}'"
  gates: []
  self_updates: true
```

`manifests/repos.yml`:

```yaml
- dest: ~/.oh-my-zsh
  url: https://github.com/ohmyzsh/ohmyzsh.git
  branch: master
  private: false
- dest: ~/.claude
  url: git@github.com:esba1ley/claude-user-settings.git
  branch: develop
  private: true          # needs SSH keys; see restore ordering
```

`dotfiles/allowlist.yml`:

```yaml
- src: ~/.zshrc
  dest: dotfiles/zshrc
- src: ~/.p10k.zsh
  dest: dotfiles/p10k.zsh
  note: regenerated wholesale by `p10k configure`
- src: ~/.condarc
  dest: dotfiles/condarc
```

`~/.claude` appears nowhere in this file. It is repo-managed.

Entries are opt-in only. A new file in `$HOME` is never captured until it is
added here by hand.

## Components

Each script does one thing, is invoked the same way, and states its
dependencies.

**`capture-*`** — read host state, write one manifest. No arguments. Idempotent
by nature: re-running produces the same file unless the host changed. Depends on
the tool it interrogates being present; exits non-zero with a clear message if
it is not.

**`restore-*`** — read one manifest, converge the host toward it. Accepts
`--dry-run`. Idempotent: running twice must leave the second run a no-op.
Checks its prerequisite first. Never removes anything the manifest omits —
convergence here is additive, because a restore that uninstalls surprises is a
restore nobody runs twice.

**`capture-dotfiles`** — copies allowlisted files, running the secret scan on
each. Refuses to write any file that trips the scan, naming the file and the
matched pattern, and exits non-zero. Does not commit.

**`restore-dotfiles`** — copies files back to their `src` locations, backing up
any existing file to `~/.config-backup/<ISO-8601 timestamp>/` first. Requires no
prerequisite and could technically run first on a bare machine, but
`RESTORE.md` orders it last; see Restore ordering for why.

**`capture-repos`** — records URL, branch, and destination for each managed
clone. Does not copy their contents.

**`restore-repos`** — reproduces each clone via `git init` / `remote add` /
`fetch` / `checkout -f`, tolerating a destination that already exists. Public
clones need no credentials; the private one fails fast with a clear message if
SSH authentication is not yet available, naming the manual step that fixes it.

**`check-drift`** — runs every `capture-*` into a temporary directory and diffs
against the committed manifests. For repo-managed clones it reports instead
whether the working tree is dirty or the branch has diverged from its origin —
a dirty `~/.claude` means personal configuration has been edited but not pushed
to its own repository, which is exactly the drift worth catching weekly.
Reports differences and exits non-zero if any are found. This is the component
sub-project B will call weekly.

**`capture`** — runs each `capture-*` in order. Convenience only; no logic of
its own.

## Safety rules

- Every `restore-*` is idempotent and supports `--dry-run`.
- `restore-dotfiles` backs up before overwriting; it never clobbers silently.
- `capture-dotfiles` refuses to write on a secret-scan hit. The gate belongs
  before the commit, not in review.
- Restore is additive. Nothing is uninstalled or deleted to match a manifest.
- The allowlist is opt-in only.

### Secret scan

Applied to every file `capture-dotfiles` would write. A hit aborts the capture
of that file and reports it. Patterns cover, at minimum: PEM private key
headers, `api[_-]?key`, `secret`, `token`, `password`, `Bearer `, AWS access key
identifiers, and long high-entropy base64 runs. False positives are expected and
acceptable — the operator decides, and an over-eager scan costs a conversation
while a missed secret costs a rotation.

### Never captured

`~/.ssh/*` and credentials or API tokens of any kind. These are restored by
hand from a password manager; `RESTORE.md` lists them.

### Captured elsewhere, deliberately

`~/.claude` is repo-managed: its configuration — `CLAUDE.md` and `rules/` —
lives in the private companion repository, not here. Its machine-local and
historical contents (`sessions/`, `history.jsonl`, `projects/`, `cache/`,
`daemon*`, `file-history/`) are neither captured nor cloned; they are history
rather than configuration, and that repository's own `.gitignore` already
excludes them.

`~/.oh-my-zsh/custom/` holds no user-authored content — only Oh My Zsh's
shipped examples and the powerlevel10k clone — so it needs no allowlist entry.
The repo-managed class reproduces both.

## Restore ordering

`RESTORE.md` documents this order, with the Apple-gated steps called out as
requiring a human:

1. macOS itself; sign in.
2. **Manual prerequisites: SSH keys from the password manager**, plus
   credentials and licences. These move early — earlier than an unthinking
   ordering would put them — because the private companion repository in step 8
   cannot be cloned without them.
3. Xcode from the App Store; accept the licence; install Command Line Tools.
4. MacPorts, then `restore-ports`.
5. miniforge, then `restore-conda`.
6. Visual Studio Code, then `restore-vscode`.
7. MacTeX, Dakota, Docker Desktop — per `vendors.yml`.
8. `restore-repos` — Oh My Zsh, powerlevel10k, and `~/.claude`. Must precede
   the next step: the captured `~/.zshrc` sets `ZSH_THEME` and sources Oh My
   Zsh, so restoring it onto a machine without those clones yields a shell that
   errors on every start.
9. `restore-dotfiles` — last, deliberately. Installers earlier in this list
   write into the very files it restores: miniforge's `conda init` appends a
   block to `~/.zshrc`, and the captured `~/.zshrc` already contains that block.
   Restoring dotfiles last makes the captured file authoritative and avoids a
   duplicated block.
10. Verification: `bin/check-drift` should report no drift against the
    manifests.

## Testing

**Framework:** bats-core (MacPorts `bats-core`, currently 1.13.0, not yet
installed — it becomes the 19th requested port and is captured by the very
manifest it tests), with `bats-assert`, `bats-support`, and `bats-file`.

bats runs the scripts as executables, so their zsh shebang is immaterial.
Internal zsh functions are not unit-tested by sourcing, which bash cannot do;
the scripts are thin wrappers over `port`, `conda`, and `code`, so black-box
testing is the correct granularity regardless.

- **Capture scripts:** round-trip — capture, restore into a temporary `$HOME`,
  diff against the original.
- **Restore scripts:** `--dry-run` assertions, and an idempotency test that runs
  the script twice and asserts the second run changes nothing.
- **`restore-repos`:** tested against a destination that already exists and is
  non-empty, since that is the real case for `~/.claude`; and for idempotency,
  where the second run must fetch and change nothing.
- **Secret scan:** fixture files containing planted synthetic secrets must be
  refused; clean fixtures must pass.
- **In-script verification** (prerequisite and post-condition checks) is plain
  zsh inside the scripts, deliberately framework-free, so it works on a bare
  machine where nothing is installed yet.
- **Full bare-metal restore** cannot be rehearsed cheaply: macOS does not
  containerise, and Docker Desktop runs Linux. It is verified against a macOS VM
  when one is available. Until then `RESTORE.md` carries an explicit
  *unverified* notice at the top rather than implying it has been proven.

A note on frameworks: the repository convention names pytest, which governs
Python code. This sub-project has none. Per-language frameworks are already the
established pattern here (Ceedling for C, gtest for C++); bats fills the shell
gap rather than contradicting anything. Should sub-project B's report generator
be written in Python, it takes pytest — a split by language, which is the normal
reason to carry two frameworks, rather than by bootstrap phase.

## Documentation changes this requires

- `.claude/ARCHITECTURE.md` — `Repository Contents` moves **Open → Frozen**,
  describing the manifest-and-dotfile structure. A new `Capture and Restore`
  aspect describes the script layer.
- `.claude/DECISIONS.md` — the 2026-09-18 entry *Defer choosing between
  documenting the host and rebuilding it* is re-marked **Superseded**, with a new
  entry recording the rebuild decision and the four choices above.
- `.claude/JOURNAL.md` — a session entry when implementation lands.

## Success criteria

1. `bin/capture` run on this machine produces manifests that describe it, and
   `bin/check-drift` immediately afterwards reports no drift.
2. Every `restore-*` script run twice leaves the second run a no-op.
3. `capture-dotfiles` refuses a file containing a planted synthetic secret.
4. A reader who has never seen this machine can follow `RESTORE.md` end to end
   without asking a question, and the document is honest about which steps a
   human must perform.
5. No secret, key, or credential appears anywhere in the repository history.
6. No personal configuration appears in this repository at all: `CLAUDE.md` and
   `rules/` are reachable only through the private companion repository, and
   `git log -p` over this repository's history contains neither.
