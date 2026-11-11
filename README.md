# OpenCode application package

This package installs [OpenCode](https://github.com/anomalyco/opencode) — the
open-source AI coding agent that lives in your terminal — as a portable Vibe
user application. It contains no binaries: each installation downloads one
pinned upstream archive and verifies it against the SHA-256 digest published in
that release's own `SHA256SUMS`.

The archives are the **standalone** CLI builds: one self-contained compiled
binary, with the JavaScript runtime linked in. Nothing outside the payload is
needed — no Node.js, no Bun, no shim:

```text
$ env -i PATH=/usr/bin:/bin HOME=$HOME ~/.vibe/opt/bin/opencode --version
opencode v2.2.7
```

```text
vibe install -g org.vibevm.opencode/opencode
opencode --version
opencode
```

The command appears in `opt/bin` of the Vibe settings directory
(`~/.vibe/opt/bin` by default) as a PATH proxy; put that directory on PATH
(`vibe self doctor --fix`), or open a shell with `vibe shell`.

| Archive | From | Digest from |
| --- | --- | --- |
| `opencode-<target>.tar.gz` / `.zip` (one compiled `opencode` binary, nothing else) | the upstream [OpenCode releases](https://github.com/anomalyco/opencode/releases) | that release's `SHA256SUMS` |

## One template, eight releases — and one concrete tag

`vibe.toml` on `main` is a **template manifest** (`[package] template = true`,
[PROP-075](https://github.com/vibevm/vibevm)): it is written once with
`{version}` and `{sha256:…}` variables plus a `[template]` declaration naming
where the versions and the digests come from, and Vibe renders it on the
client, at resolution time, for whichever version is asked for. One file covers
the whole version domain, and no digest is ever transcribed by hand.

`[package].version` (`0.0.1`) is the **template revision**, not an OpenCode
version. It is numbered below every produced version on purpose — see
*Publishing* below.

### Why the versions are listed and not discovered

`[template.versions] kind = "list"`, one row per version, and **not**
`kind = "json"` over the GitHub releases API. That is a measured decision, not
a preference:

- `[template.versions].url` must be a credential-free HTTPS URL with **no query
  string** — the manifest validator refuses `?`. So the only reachable document
  is `https://api.github.com/repos/anomalyco/opencode/releases`, with GitHub's
  default page size of 30 and no way to ask for more.
- That repository releases **three things from one tag space** — `vX.Y.Z` (the
  CLI *and* the desktop app together), `process-daemon-vX.Y.Z` (a separate
  daemon) and `pr-*` preview tags — and a JSON Pointer selector cannot filter.
- Measured on 2026-11-11: the default page held 30 releases, of which 26 were
  `vX.Y.Z` but only **8** were 2.x (the rest 1.18/1.19), spanning **74 days**,
  in **2.07 MB** of JSON per resolution.

A discovered domain would therefore *drop* each version about two months after
its release — including a version pinned in a corporate drop — and would offer
the 1.x line this package does not cover. A listed domain keeps every listed
version installable for good. The price is one row per upstream release; the
digests still come from upstream, so a release costs one line and no checksum
transcription.

The domain is the 2.x line: **2.2.0 … 2.2.7**. Every one of those releases was
checked to publish all twelve CLI archives and a `SHA256SUMS` naming them.

### Excluded on purpose

Two families of upstream artifact are **not** part of this package:

- `opencode-desktop-*` (`.deb`, `.rpm`, `.AppImage`, `.dmg`, `.exe`,
  `.app.tar.gz`, `.zip`) — the OpenCode **desktop application**, a different
  product that happens to ship from the same release tag. Twelve such assets
  sit in the 2.2.7 release and none is installed here.
- the `process-daemon-vX.Y.Z` releases — a separate daemon with its own tag
  line and its own version numbers, not the CLI.

## Targets

| Installer option | `--platform` / `--arch` | Upstream asset |
| --- | --- | --- |
| `linux-x64` | `linux` / `x86_64` | `opencode-linux-x64.tar.gz` |
| `linux-arm64` | `linux` / `aarch64` | `opencode-linux-arm64.tar.gz` |
| `linux-musl-x64` | `linux-musl` / `x86_64` | `opencode-linux-x64-musl.tar.gz` |
| `linux-musl-arm64` | `linux-musl` / `aarch64` | `opencode-linux-arm64-musl.tar.gz` |
| `linux-x64-baseline` | `linux-baseline` / `x86_64` | `opencode-linux-x64-baseline.tar.gz` |
| `linux-musl-x64-baseline` | `linux-musl-baseline` / `x86_64` | `opencode-linux-x64-baseline-musl.tar.gz` |
| `macos-x64` | `macos` / `x86_64` | `opencode-darwin-x64.zip` |
| `macos-arm64` | `macos` / `aarch64` | `opencode-darwin-arm64.zip` |
| `macos-x64-baseline` | `macos-baseline` / `x86_64` | `opencode-darwin-x64-baseline.zip` |
| `windows-x64` | `windows` / `x86_64` | `opencode-windows-x64.zip` |
| `windows-arm64` | `windows` / `aarch64` | `opencode-windows-arm64.zip` |
| `windows-x64-baseline` | `windows-baseline` / `x86_64` | `opencode-windows-x64-baseline.zip` |

Vibe picks the option of the host's platform and architecture and asks when it
cannot decide. The musl rows carry the platform id `linux-musl`, the same
spelling `org.vibevm.codkit/codkit` uses for its `linux-musl-x64`: a musl
system is detected as `linux`, so it is asked for by name
(`--platform linux-musl`).

Every archive holds **exactly one entry** — the file `opencode`
(`opencode.exe` on Windows) at the archive root, with no enclosing directory,
no symbolic links and no special entries. There is therefore no `payload_root`,
an ordinary `archive` step places it, and the launcher's `executable` is the
bare file name.

### The `baseline` builds, and why they are a platform of their own

Upstream publishes `baseline` archives for x86-64 CPUs **without AVX2** —
pre-Haswell Intel and pre-Excavator AMD. They are the same product, built for
an older instruction-set baseline, and on a modern CPU the ordinary build is
the one you want.

They are exposed as four installer options the user asks for by name, on
platform ids of their own: `linux-baseline`, `linux-musl-baseline`,
`macos-baseline`, `windows-baseline`.

```text
vibe install -g org.vibevm.opencode/opencode --platform linux-baseline
```

The alternative — a second option on `linux` / `x86_64` — was rejected for a
mechanical reason, not a stylistic one. `detected_platform()` only ever answers
`linux`, `macos` or `windows` (PROP-059 `#installer-options`), so two options
on one platform/arch pair are *ambiguous*: an attended install would ask
every user on every machine which of the two to take, and an **unattended** one
would refuse outright with `installer selection required for linux/x86_64`.
A platform id of its own keeps the baseline rows fully installable while a
plain `vibe install -g org.vibevm.opencode/opencode` stays a single
unambiguous answer. Vibe never auto-selects a baseline build, which is exactly
the intent: AVX2 cannot be detected at planning time, and a user who needs the
baseline knows it.

`vibe app installers org.vibevm.opencode/opencode` lists all twelve, grouped by
platform.

## How the launcher runs

Each option declares one launcher, `opencode`, and exposes it as the public
command `opencode`. The launcher runs the payload binary directly; there is
nothing else in the payload to run.

## Self-update, and why a payload install cannot do it

OpenCode checks for new versions on startup and can install them. Both halves
matter in a closed network, and they behave differently here.

**It cannot replace this installation.** Before updating, OpenCode detects
*how* it was installed: `$HOME/.opencode/bin/opencode` (the `curl` installer), a
Homebrew Cellar path, a winget path, or an npm/pnpm/bun/yarn-managed
`@opencode/cli` package. A binary unpacked into a Vibe payload is none of
those, so the detection finds nothing and the update refuses. Measured against
2.2.7, with the payload binary on a bare PATH:

```text
$ env -i PATH=/usr/bin:/bin HOME=… opencode upgrade
┌  Upgrade
│
■  Could not detect the installation method. Pass --method to choose how to upgrade OpenCode.
│
└  Failed
```

The Vibe-owned payload is therefore never rewritten behind Vibe's back, and
`vibe app` stays the only thing that changes the installed version.

**The version *check* is on by default** and is one HTTPS GET to
`https://opencode.ai/update/api/…` at startup, wrapped in a handler that logs
the failure and continues. In a closed network it simply fails. To switch it
off there are exactly two documented switches, both read by the compiled 2.2.7
binary:

- `OPENCODE_DISABLE_AUTOUPDATE=1` (also `true`, case-insensitive) — tested
  *before* any configuration file is read and before any request is made.
- `"update": "disable"` in the **global** configuration (`config.json`,
  `opencode.json` or `opencode.jsonc` in OpenCode's config directory, or
  `/etc/opencode/opencode.json`, which upstream documents as overriding all of
  them). The accepted values are `"disable"`, `"notify"` (the default) and
  `"auto"`; a project-level value is ignored.

**This package sets neither, deliberately.** A Vibe launcher has no `env`
table — `[[installer_options.launchers]]` carries `id`, `name`, `executable`,
`kind`, `args`, `working_directory` and the menu fields, and nothing else
(PROP-059 `#launch-points`) — so there is no way to scope an environment
variable to this command. The one declarative facility that exists,
a `kind = "env"` step, changes the **user's durable environment** machine-wide
(PROP-065 `#env-step`): it would silently retune every other OpenCode on that
account, including one the user installed themselves, which is not a
packaging decision to take on their behalf.

A closed-network site turns the check off once, for the whole machine, in the
place that already holds such policy — the environment the site profile
establishes, or `/etc/opencode/opencode.json`:

```json
{ "update": "disable" }
```

## The prompt bases, and why this package carries them

`prompt-bases/<opencode-version>/` holds OpenCode's **own** system-prompt text,
vendored verbatim from the upstream tag of that exact version, with a
`PROVENANCE.toml` saying where every byte came from and what has been proved
about it. That is unusual for a package whose job is to download a binary, so
here is the whole reason.

Vibe can put a project's boot lane — its standing instructions, composed from
the project's own boot files — into the coding agent's **system prompt**, so
the provider caches it once and every later session in that project reads it
from the cache instead of re-reading files with tools
([PROP-076](https://github.com/vibevm/vibevm)). For that to be safe, a harness
needs an instruction slot that is **additive** (it adds to the harness's own
prompt rather than replacing it), **file-based** (a ~300 KB lane cannot go on a
command line), **per-launch** (it must not leak into other sessions), and
**not inherited by subagents** (a bounded worker must not receive a central
prefix). Claude Code and Codex each have one, so vibe passes them a flag and is
done.

OpenCode has no such slot. Its configuration `instructions` array is additive
and file-based — and a `subagent` child session receives it, measured on 2.2.7:
the child's request came back byte-identical to the primary's with the lane in
it. Its only **subagent-safe** slot is an agent's `system`, and that one
*replaces* OpenCode's base prompt instead of adding to it. So the only way to
give an OpenCode session a boot lane without silently deleting the harness's
own instructions is for vibe to supply **OpenCode's own base prompt for the
exact installed version**, with the lane after it.

That is what these files are for, and it is why they are versioned data with
provenance rather than text in vibe's source: a base from the wrong version, or
for the wrong model family, produces a session whose harness instructions are
not the ones OpenCode wrote, with no error anywhere — which is worse than
delivering no lane at all. Carrying them **in this package** means a new
OpenCode version ships its bases with the package and needs no vibe release,
and a closed network receives them through the same mirror or drop that already
carries the package.

Design authority: `PROP-076#SYS-SUPPLIED-BASE` and
`campaigns/sys-prompt-2026-11/HARNESS-DELIVERY.md` §5 ("OpenCode, second
look") in the vibevm repository. The runbook for the next version is under
*Maintaining* below.

## Closed networks

The same package, unedited, installs where `github.com` cannot be reached. The
administrator mirrors it into the corporate registry and the archives land in
the machine's `[artifacts] packages` drop; every archive is still verified
against the digest from upstream's own `SHA256SUMS`.

Because the manifest is a template, the closed network also has to reach the
one document it declares per version — that release's `SHA256SUMS`. The mirror
does that once, on the connected side, and carries the result into the
registry as a **template snapshot** (PROP-075 `#MIRROR-TEMPLATE-SNAPSHOT`); the
client then renders locally from the snapshot with the same code as outside and
makes no request at all. `payloads = true` makes the same run fill the drop.

1. On a connected machine, with a `mirror.toml`:

   ```toml
   schema = 1

   [select]
   packages = ["org.vibevm.opencode/opencode"]

   [[template]]
   package = "org.vibevm.opencode/opencode"
   versions = ["=2.2.7"]
   payloads = true

   [payloads]
   into = "/srv/vibe-drop"
   max_total = "2 GiB"
   ```

   ```text
   vibe registry mirror vibespecs --into <corporate monorepo url> --config mirror.toml
   ```

   That one run copies the template package, snapshots the digest document of
   the selected version, and downloads all twelve archives into the drop at

   ```text
   <drop>/org.vibevm.opencode/opencode/2.2.7/opencode-linux-x64.tar.gz
   <drop>/org.vibevm.opencode/opencode/2.2.7/opencode-windows-x64.zip
   …
   ```

   `vibe artifacts where org.vibevm.opencode/opencode` prints, for every
   installer option, the exact file name, digest and destination path, and
   whether a verified copy is already there.

2. A site profile names the registry, the drop and the source policy:

   ```toml
   schema = 1
   name = "example-site"

   [[registry]]
   name = "corp"
   type = "monorepo"
   url = "ssh://git.example.internal/vibe/registry.git"

   [source_policy]
   exclude_hosting = ["github", "gitverse"]

   [artifacts]
   packages = "/srv/vibe-drop"
   ```

   applied with `vibe self apply site.toml`; then
   `vibe install -g org.vibevm.opencode/opencode` reads the package and the
   snapshot from `corp` and the archive from the drop, with no connection to
   any public host.

**Without a mirror snapshot a cold machine cannot render the template
offline.** A template's digest document is a real input: a fresh home with the
package in a `localdir` registry and the archive in the drop still refuses,
naming the document and the cache
(`spec://org.vibevm.core/vibevm/common/PROP-075#TEMPLATE-CACHE`). Either mirror
the template (above), or let the machine render the version once while it has
access — after that `<settings>/cache/templates/org.vibevm.opencode/opencode/`
holds the immutable digests and `--offline` resolves from them with no request.
A concretely published version (see below) needs no document at all.

With a proxy instead of a drop, the archives and the digest document are
downloaded through the `[proxy]` the site profile declares (`mode = "px"` with
a PAC file is supported).

## Publishing

The registry resolves a per-package repository by its **tag list**, and
rendering a template fetched that way is the one part of PROP-075
`#TEMPLATE-DISCOVERY` that is not wired yet (`#P2-AS-BUILT` (8)): such a client
reads the template's own revision as if it were a release and installs a
manifest whose `{version}` was never substituted. Hence the arrangement, which
is the one `org.vibevm.node/node` uses:

- **`main`, tag `v0.0.1`** — the template. It is what a directory-shaped
  registry resolves: `--registry DIR`, a `localdir` source, a `monorepo`
  working copy, a mirrored layout registry. That covers the whole corporate
  path, where every version in the snapshot is installable from one manifest.
- **tag `v2.2.7`** — the same manifest **rendered concretely** for that one
  version, by Vibe's own renderer (`crates/vibe-template`), on the branch
  `release/2.2.7`. This is the tag the public index carries and the one a
  public `vibe install -g org.vibevm.opencode/opencode` resolves.
- The revision is numbered **below** every produced version (`0.0.1` <
  `2.2.0`) so that even a client that ignores the index's `template` flag
  prefers a real release over the template.

A concrete tag is produced, never hand-written:

```text
vibe install -g org.vibevm.opencode/opencode --registry <dir holding this package>
cp <settings>/cache/templates/org.vibevm.opencode/opencode/rendered/v<version>/vibe.toml .
```

## Maintaining

A new upstream release needs **one row**:

```toml
[[template.versions.item]]
version = "2.2.8"
```

Then, in order:

1. Check the release really carries the twelve CLI archives and a `SHA256SUMS`
   naming them — `gh release view v2.2.8 -R anomalyco/opencode --json assets`.
2. `vibe app versions org.vibevm.opencode/opencode --registry <dir>` — the new
   version must appear and render.
3. **Put the version's prompt-base set in place first** — see *Adding prompt
   bases for a new OpenCode version* below. A concrete tag is cut only after
   its `prompt-bases/<version>/` directory exists, because vibe reads the
   bases from the installed version's own package tree and the public path
   resolves the concrete tag: a concrete tag without them is a version whose
   lane cannot be delivered. (`v2.2.7` was re-cut on 2026-11-11 for exactly
   this reason, the day it was first published and before it had users.)
4. Bump `[package].version` (the template revision) and tag it.
5. Render the concrete manifest for the new version, commit it on
   `release/2.2.8`, tag `v2.2.8`, and reindex.

What has to be checked when upstream changes *shape* rather than merely
publishing a release:

- a new target (a new archive in `SHA256SUMS`) — a new `[[installer_options]]`
  row, `template_optional = true` until every listed version has it;
- a target upstream stops building — the row becomes `template_optional`
  rather than disappearing, so older versions keep installing;
- the archive gaining a second entry or an enclosing directory — today there is
  none, so the manifest declares no `payload_root` and the launcher names the
  bare binary; both would have to change;
- a release whose archive set is incomplete — leave it out of the list rather
  than letting it render half a manifest;
- a new `desktop` or daemon asset naming scheme — the exclusions above are by
  prefix, so check that nothing new slips into the CLI set.

Upstream publishes digests but **no sizes**, so no step declares `size`: the
step panel falls back to the server's `Content-Length` and a mirror's byte
total is reported as a lower bound.

A note on upstream's 2.2.7 digests: `opencode-linux-x64.tar.gz` and
`opencode-linux-x64-baseline.tar.gz` are byte-identical in that release, and so
are the two `-musl` ones. That is upstream's business; each option names its
own asset and takes that asset's digest, so nothing here depends on their
being different.

### Adding prompt bases for a new OpenCode version

Written for whoever — person or agent — opens this repository next. The
previous set is the worked example: read `prompt-bases/2.2.7/PROVENANCE.toml`
first, because every field below exists in it already.

**1. Where to look.** Upstream is `https://github.com/anomalyco/opencode`
(`sst/opencode` 301-redirects there). Find the tag matching the version you are
packaging — `v2.2.8` for OpenCode 2.2.8 — and record its **commit**, not just
the tag, since a tag can move. Read the commit's source tarball; do **not**
read the published npm artifact or the compiled binary, which inline these
files through a bundler.

**2. Fetch the seven base files**, at these paths, byte-exact — every blank
line, every non-ASCII character, the trailing newline:

```text
packages/core/src/session/runner/prompt/system.txt      the generic base
packages/core/src/plugin/system-prompt/anthropic.txt    appended for claude*
packages/core/src/plugin/system-prompt/gpt.txt          replaces, for gpt*
packages/core/src/plugin/system-prompt/gpt-astra.txt    replaces, for gpt*astra*
packages/core/src/plugin/system-prompt/kimi.txt         replaces, for kimi*
packages/core/src/plugin/system-prompt/meta.txt         replaces, for muse*
packages/core/src/plugin/system-prompt/trinity.txt      replaces, for trinity*
```

Record each one's byte length, line count and sha256 and **verify what you
wrote to disk against them**. Transcription through any chat or tool is where
bytes get eaten: in 2.2.7 the hazards were a double blank line in
`gpt-astra.txt`, six non-ASCII codepoints across `gpt-astra.txt` and
`kimi.txt`, and three lines consisting of a single `>` in `kimi.txt`.

**3. Carry the licence.** Fetch the tag's root `LICENSE` byte-exact into
`prompt-bases/<version>/LICENSE` and record its digest in `PROVENANCE.toml`
under `[upstream.license]`. It lives **inside the version directory**, not
beside it, so each set is self-contained: one directory you can copy, audit or
drop carries both the bytes and the licence that governs them, and a version
whose upstream licence changes records its own. See *Licences* below.

**4. Record the assembly files and what each decides.** These are **drift
tripwires**: `PROVENANCE.toml` carries a sha256 of each, and the sync tool
refuses to verify a version where any of them changed. The ones that decide the
most:

| File | What it decides |
| --- | --- |
| `packages/core/src/plugin/optimize.ts` | the model→base selection table, each family's append/override mode, and the bail-out that leaves an agent's own `system` untouched |
| `packages/core/src/session/system-prompt.ts` | the render rule: the `${OPENCODE_TOOL_GUIDANCE}` token, its three gate tool names (`shell`, `write`, `edit`), the bullet strings and their order |
| `packages/core/src/session/model-request.ts` | the system-part array, and that an agent's `system` is used **raw** rather than rendered |
| `packages/core/src/config/variable.ts` | `{file:}` substitution — including the whitespace **trim** the lane hand-over depends on |
| `packages/core/src/config.ts` | where substitution is applied, including to `OPENCODE_CONFIG_CONTENT` |
| `packages/core/src/config/plugin/instruction.ts` | the `AGENTS.md` walk and the ordering of config `instructions` |
| `packages/core/src/session/context.ts` | the order of instruction sources inside the joined part |
| `packages/core/src/instructions/builtins.ts` | the date and `<env>` block — i.e. where the per-day text sits relative to the lane |
| `packages/core/src/tool/plugin/subagent.ts` | how a child session is built, and therefore worker exclusion |
| `packages/core/src/plugin/identity.ts`, `tool/plugin/opencode.ts`, `plugin/internal.ts`, `plugin/hooks.ts` | the other system parts and the order the hooks add them in |

The full list — 35 paths, each with its `decides` line — is in
`prompt-bases/2.2.7/PROVENANCE.toml`. Copy it forward and re-digest.

**5. Run the sync tool** (in the vibevm repository):

```text
cargo run -p xtask -- opencode-bases --version 2.2.8
```

Its two outcomes mean different things:

- **fingerprints unchanged** → it fetches the seven files, writes the
  directory, runs the capture proof against the installed 2.2.8 binary, and
  sets `verified = true` with the proof's id and date. That is the only thing
  that may set `verified`.
- **fingerprints changed** → it **refuses** to verify and prints a diff of the
  changed assembly files. Read the diff and answer one question: does the
  change affect the composition this provenance describes? If it only adds a
  rule the recorded data can express (another `[[select]]` row, another
  `[[render.bullet]]`), extend `PROVENANCE.toml` and re-run. If it needs a rule
  the interpreter cannot express, **stop**: extend the interpreter in vibe
  first, with its own spec amendment. Never hand-edit `verified = true` — a
  guessed base is the failure this whole mechanism exists to prevent.

**6. The six verification conditions**, if you run them by hand. The scripts
live in the vibevm repository under `campaigns/sys-prompt-2026-11/live/`:
`make-sample.sh` builds the sample project, `opencode-capture.sh` captures a
plain and a carrier run, and `opencode-subagent-capture.sh` with
`fake-provider-subagent.py` drives a real child session offline so worker
exclusion can be measured rather than argued. The conditions are listed in
`PROP-076#SYS-SUPPLIED-BASE`: the reproduced base equal to the installed
program's own; one contiguous insertion and nothing else changed; the inserted
bytes equal to the lane under the named transformation only; the seam shown
both ways; prefix stability across two launches; a child carrying the lane zero
times; and one live check that an instruction existing **only** in the base is
still obeyed. Live checks use the local model and the small sample — a 202 KB
lane is proved offline, where it costs nothing.

**7. Then the ordinary release steps** of this section: the
`[[template.versions.item]]` row, `vibe app versions`, the template revision
bump, the rendered concrete manifest on `release/<version>`, the tag, and the
reindex. A prompt-base directory is part of the template tree, so it travels
with both tags.

**8. Refresh the mirror and the drop.** The bases are ordinary files in the
package tree, so a `vibe registry mirror` run that copies the package copies
them too — nothing new is needed in `mirror.toml`. The **drop** needs one
thing the archives do not provide: upstream's standalone archive holds the
compiled binary alone with no licence file beside it, so the drop should carry
`LICENSE` next to the archives for the version it pins. `prompt-bases/<version>/LICENSE`
in this repository is the byte-exact copy to use.

## Licences

Three layers, and they are not the same licence. Saying so once here is cheaper
than a reader guessing.

**1. The packaging — UPL-1.0.** This repository's own work: `vibe.toml`, the
documents, the boot snippet, and the provenance records
(`prompt-bases/*/PROVENANCE.toml`). See `LICENSE.md`. `[package].license` in
the manifest is deliberately **not** this: it names the licence of what an
installation puts on the machine.

**2. The OpenCode binaries — MIT, upstream's, and this repository never
redistributes them.** Each installer option names an upstream URL and a SHA-256
from that release's own `SHA256SUMS`; the bytes travel from upstream to the
machine, and `[package].license = "MIT"` records what they are. The standalone
archive carries the compiled binary **alone, with no licence file beside it**,
so a corporate drop — which *does* redistribute the archives — should carry
upstream's `LICENSE` next to them; `prompt-bases/<version>/LICENSE` here is the
byte-exact copy for that purpose.

**3. The vendored prompt-base text — MIT, `Copyright (c) 2025 opencode`.**
`prompt-bases/<version>/*.txt` is OpenCode's own system-prompt text, copied
verbatim from the upstream tag, and it is the one place where this repository
redistributes someone else's work. Upstream's licence travels with it, byte-exact,
at `prompt-bases/<version>/LICENSE`, which is what MIT asks for: "The above
copyright notice and this permission notice shall be included in all copies or
substantial portions of the Software." The upstream path, commit and digest of
every file, and of the licence itself, are recorded in that version's
`PROVENANCE.toml`. There is no `SPDX-License-Identifier` string in upstream's
licence file, no per-file header on the prompt files, and no `NOTICE` governing
them.

Any other harness package that comes to carry upstream prompt text follows this
same pattern — the licence of the text it vendors, byte-exact, inside the
version directory, with its digest in the provenance record. For Qwen Code that
licence is Apache-2.0, which also means carrying any `NOTICE` upstream ships.
