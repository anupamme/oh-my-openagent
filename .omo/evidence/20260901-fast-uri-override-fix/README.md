# fast-uri override fix (CVE-2026-13676 review follow-up)

## What was tested

PR #6654 / branch `fix-repo-oh-my-openagent-cve-2026-13676-fast-uri` (commit
`fa9c19f04`) added `fast-uri@4.1.2` as a new, unused direct dependency but
left the root `overrides.fast-uri: "^3.1.2"` entry unchanged, so the
vulnerable resolution survived. This revision instead:

- Removes the unused `"fast-uri": "4.1.2"` line from root `package.json`
  `dependencies`.
- Changes `overrides.fast-uri` from `"^3.1.2"` to `"3.1.6"` (exact pin, the
  latest 3.x release — not just the reviewer's literal `3.1.3`, see rationale
  below).
- Regenerates `bun.lock` for real via `bun install` (bun 1.4.0, matching the
  CI-authoritative pinned version per `AGENTS.md`/`CLAUDE.md`).

Commands run: `bun install`, `bun pm why fast-uri` (before/after),
`grep -n '"fast-uri' bun.lock` (before/after), `trivy fs --scanners vuln .`
(before upgrading trivy 0.58.1 → 0.74.0, and after), `bun run typecheck`,
`bun test`, `bun run build`.

## Why 3.1.6, not the reviewer's literal 3.1.3

The review comment suggested pinning to exactly `3.1.3`. A Trivy scan run
during this fix showed the vulnerable `fast-uri@3.1.2` resolution is flagged
for **three** CVEs, not just the one this branch is named after:

- CVE-2026-13676 — patched at 3.1.3+ (the CVE this branch targets)
- CVE-2026-16221 — patched at 3.1.4+
- CVE-2026-18446 — patched at 3.1.5+

Pinning to exactly `3.1.3` would have cleared only the first and left the
other two present against the same package. `3.1.6` is the latest release on
the `fast-uri` 3.x line (confirmed via `bun pm view fast-uri versions`) and
clears all three without a major-version bump, consistent with the
reviewer's stated intent to avoid jumping to the 4.x line.

## What was observed

**Dependency tree** (`bun-pm-why-fast-uri-before.txt` /
`-after.txt`): before, every consumer (`ajv@8.20.0` requiring `^3.0.1`,
`@code-yeongyu/senpi` pinning `3.1.4` exactly, and root's unused `4.1.2`
entry) was forced down onto a single `fast-uri@3.1.2` by the override —
proving the override, not the direct dependency, is the real control point.
After, the same consumers (plus `ajv-formats`, `@modelcontextprotocol/sdk`
chains) all collapse onto a single `fast-uri@3.1.6`.

**Lockfile** (`bun-lock-fast-uri-grep-before.txt` / `-after.txt`): before,
`bun.lock` had one resolved `packages["fast-uri"]` entry
(`fast-uri@3.1.2`) plus a *separate*, never-installed `"fast-uri": "4.1.2"`
line under the root workspace's declared `dependencies` (proof the original
fix's lockfile edit was hand-written, not a real `bun install` — there was no
resolved `fast-uri@4.1.2` package entry anywhere). After, there is exactly
one resolved entry, `fast-uri@3.1.6`, and no `4.1.2`/`3.1.2`/other version
string remains anywhere in the file.

**CVE scan** (`trivy-scan-after.json`, `trivy-scan-after-summary.txt`,
`trivy-version.txt`): the locally available Trivy (0.58.1) does not parse
Bun's lockfile format at all — a `trivy fs --scanners vuln .` run against
the unfixed tree only scanned two `package-lock.json` files and one vendored
`pnpm-lock.yaml`, never the root `bun.lock`. Trivy was upgraded to the
current stable release (0.74.0), which added a `[bun]` language analyzer and
does scan `bun.lock` directly. Re-running after the fix:
- Target `bun.lock` (type `bun`): 21 total vulnerabilities, **0** naming
  `fast-uri`. CVE-2026-13676 / CVE-2026-16221 / CVE-2026-18446 are absent
  from the root project's dependency graph.
- Target `packages/shared-skills/upstreams/open-design/pnpm-lock.yaml`
  (type `pnpm`): still shows `fast-uri@3.1.2` flagged for the same three
  CVEs. **This is a known, explicitly out-of-scope finding** — see below.

**Full validation matrix**: `typecheck.log` — clean, no errors, all
workspace packages typechecked. `build.log` — `bun run build` completed
("build: all steps completed"), `dist/index.js` rebuilt. `bun-test.log` —
16552 pass, 8 skip, 1 fail, 46483 expect() calls, 16561 tests across 2136
files. The 1 failure
(`scan tool: pinned real-binary fixtures > ...it is the 0.43.0 OMO pin`,
`packages/ast-grep-mcp/src/tools/scan.test.ts:203`) is a machine-environment
fixture check for a locally provisioned `ast-grep` binary under
`~/.omo/runtime/ast-grep/`, which was never provisioned on this machine
(`~/.omo/runtime` does not exist here). Confirmed pre-existing and unrelated
to this change by stashing this fix out (back to commit `fa9c19f04`) and
re-running just that test file: it fails identically
(`existsSync(PINNED_SG_PATH)` → `false`) with or without the fast-uri fix.

**Compatibility impact**: `3.1.2 → 3.1.6` is a patch-level bump within the
`fast-uri` 3.x line — no major-version change, no API surface change. `ajv`
(`^3.0.1`) and senpi's pinned `3.1.4` requirement are both satisfied by
`3.1.6` under the override. No consumer needed code changes; this is
confirmed by the green typecheck/build and the test suite showing no
newly-introduced failures.

## Why it is enough

The override is the single point controlling every `fast-uri` resolution in
the root project's dependency graph (proven by `bun pm why` before/after).
Regenerating the lockfile via a real `bun install` (not a hand-edit) and
confirming a single `3.1.6` entry, backed by a scanner that can now actually
parse this repo's lockfile format and reports zero `fast-uri` findings on
that target, is a materially stronger proof than the original PR's approach
of adding an unused dependency. The full typecheck/test/build matrix confirms
no regression from the version bump.

## What was omitted / explicitly out of scope

`packages/shared-skills/upstreams/open-design` is a vendored git submodule
(`nexu-io/open-design`, shallow-cloned, pinned at a fixed upstream commit).
Its own `pnpm-lock.yaml` independently pins `fast-uri: 3.1.2` and is flagged
by Trivy for the same three CVEs. This is **not fixed by this change** and
cannot be fixed by editing this repo's root `package.json`/`bun.lock` — the
submodule is a separate, third-party-controlled repository. Remediating it
would require bumping the submodule pointer to a commit/release with a
patched `fast-uri`, which is an unrelated, separate change. This is called
out here as residual/known risk rather than silently left out of the scan
summary.

No secrets, tokens, or credential-bearing output were produced by any of the
commands above; nothing was redacted.
