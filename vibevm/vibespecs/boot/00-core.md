# OpenCode application package

This package declares how Vibe installs OpenCode, the official **standalone**
upstream CLI build, as a portable user application on twelve targets. It
contains no binaries. Each installer option downloads one archive identified by
its exact URL and SHA-256, unpacks it, and declares one launcher, `opencode`,
exposed as the public command `opencode`.

Every archive holds exactly one entry — `opencode` (`opencode.exe` on Windows)
at the archive root, with no enclosing directory and no symbolic or special
entries. So no option declares `payload_root`, the launcher's `executable` is
the bare file name, and an ordinary `archive` step places it. A self-contained
compiled binary needs nothing on PATH: verified by running `opencode --version`
under `env -i PATH=/usr/bin:/bin`, which prints `opencode v2.2.7` and exits 0.

**`vibe.toml` is a template manifest** (`[package] template = true`,
PROP-075): one file with `{version}` and `{sha256:…}` variables plus a
`[template]` declaration, rendered on the client at resolution time. The
digests come from each release's own `SHA256SUMS`
(`https://github.com/anomalyco/opencode/releases/download/v{version}/SHA256SUMS`),
which carries no sizes — so no step declares `size`, the same deliberate loss
the Node.js package records.

`[template.versions] kind = "list"`, not `kind = "json"`, for one measured
reason: `[template.versions].url` must carry **no query string** (the validator
refuses `?`), so the only reachable document is the GitHub releases API at its
default page size of 30; that repository releases the CLI and the desktop app
from one `vX.Y.Z` tag space, plus `process-daemon-v*` and `pr-*` tags, and a
selector cannot filter. Measured 2026-11-11: 30 releases → 26 `vX.Y.Z`, of
which only **8** are 2.x, spanning 74 days, 2.07 MB per resolution. A
discovered domain would drop every version about two months after release,
including one pinned in a corporate drop, and would offer the 1.x line this
package does not cover. **Do not convert this to `kind = "json"`** without
re-measuring that page, and never to buy brevity.

`[package].version` is the **template revision** (`0.0.1`), not an OpenCode
version, and it is deliberately **below** every produced version. Rendering a
template fetched from a per-package Git registry is not wired (PROP-075
`#P2-AS-BUILT` (8)), so such a client reads the revision as a release and would
install an unsubstituted `{version}` manifest; numbered below `2.2.0`, it
always loses to a concretely published version. Keep it that way: a revision
bump goes `0.0.1` → `0.0.2`, never to `1.0.0`.

Two tags therefore mean two different things, the arrangement
`org.vibevm.node/node` uses:

- `v0.0.1` on `main` — the template, which every directory-shaped registry
  resolves (`--registry DIR`, `localdir`, a `monorepo` working copy, a mirrored
  layout registry). This is the whole corporate path.
- `v2.2.7` on `release/2.2.7` — the same manifest rendered concretely by
  Vibe's own renderer (`crates/vibe-template`), which is what the public index
  carries and what a public `vibe install` resolves. Never hand-write it: take
  `<settings>/cache/templates/org.vibevm.opencode/opencode/rendered/v<version>/vibe.toml`
  after one install from a local registry.

**The `baseline` options must keep a platform id of their own.** Upstream's
`baseline` archives are built for x86-64 CPUs without AVX2. They are exposed as
`linux-baseline`, `linux-musl-baseline`, `macos-baseline` and
`windows-baseline` — not as second options on `linux` / `macos` / `windows` —
because `detected_platform()` only ever answers `linux`, `macos` or `windows`
(PROP-059 `#installer-options`): two options on one platform/arch pair make an
attended install ask every user on every machine and make an **unattended** one
refuse with `installer selection required`. The baseline rows must stay
selectable (`--platform linux-baseline`) and never selected. The `linux-musl`
rows follow `org.vibevm.codkit/codkit`'s precedent.

Two upstream artifact families are excluded on purpose and must stay excluded:
every `opencode-desktop-*` asset (the desktop application, a different product
shipped from the same release tag) and the `process-daemon-vX.Y.Z` releases (a
separate daemon with its own tag line).

**Self-update.** A payload install cannot replace itself: OpenCode detects its
installation method (`$HOME/.opencode/bin`, Homebrew Cellar, winget, or an
npm-managed `@opencode/cli`), finds none, and refuses — verified, `opencode
upgrade` prints `Could not detect the installation method`. The startup version
*check* is a separate thing and is on by default (`update = "notify"`): one
HTTPS GET to `https://opencode.ai/update/api/…`, which simply fails in a closed
network. The package sets no switch for it, deliberately: a launcher has no
`env` table (PROP-059 `#launch-points`) and the only declarative facility, a
`kind = "env"` step, changes the user's durable environment machine-wide
(PROP-065 `#env-step`) and would retune every other OpenCode on that account.
`README.md` documents the two real switches — `OPENCODE_DISABLE_AUTOUPDATE=1`
and `"update": "disable"` in the global or `/etc/opencode` configuration — for
the site to set once. **Do not add an `env` step or write into any user's
OpenCode configuration from this package.**

Before changing the package, read `vibe.toml` and `README.md`. **Do not add a
generator**: the renderer is Vibe's own pure one. A new upstream release is one
`[[template.versions.item]]` row plus a revision bump plus a concrete tag —
and first a check that the release actually publishes all twelve CLI archives
and a `SHA256SUMS` naming them, because a listed version that cannot render is
a refusal at install time.

Upstream is MIT; this package's own files are UPL-1.0. Both are stated in
`README.md` and must stay stated.
