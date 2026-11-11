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
3. Bump `[package].version` (the template revision) and tag it.
4. Render the concrete manifest for the new version, commit it on
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

## License

This package — its manifest and its documents — is UPL-1.0; see `LICENSE.md`.

OpenCode itself is **not**: it is distributed under the **MIT License**.
`[package].license` in the manifest names the license of what an installation
puts on the machine, `MIT`. The standalone archive carries the compiled binary
alone, with no license file beside it; upstream's text is in the repository the
`homepage` names.
