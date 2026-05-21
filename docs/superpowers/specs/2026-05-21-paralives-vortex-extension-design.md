# Paralives Vortex Extension — Design

**Date:** 2026-05-21
**Status:** Draft / approved for plan-writing
**Target:** Paralives Early Access (May 25, 2026), Steam App ID `1118520`

## Summary

Build a Vortex (Nexus Mods mod manager) game extension for Paralives, using
[`Nexus-Mods/game-description-language`](https://github.com/Nexus-Mods/game-description-language)
(GDL) as a git submodule. v1 is a minimal **file overlay** extension: any
archive a user drops into Vortex is deployed by hardlink into the Paralives
install directory, with Vortex handling conflict resolution and uninstall.

This is intentionally a thin slice. The exact mod surface for Paralives
(AppData CC mods, in-game-authored feature mods, community script mods) is
not publicly documented, and EA ships in four days. Shipping a permissive
overlay extension on day one and iterating once the actual mod ecosystem
appears is preferred over front-loading guesses.

## Goals

- A Vortex extension that auto-discovers a Steam install of Paralives.
- Generic single-mod-type deployment: archive contents → game install root.
- Distributable via GitHub Releases and Nexus Mods (site mod listing).
- Maximize declarative YAML, minimize hand-written TS, so the extension is
  easy for others to read, fork, and extend post-EA.
- Foundation that can grow into AppData-aware mod types, manifest parsing,
  toolbar actions, etc., without rewrites.

## Non-goals

- Steam Workshop subscription awareness or deduplication.
- Parsing or validating Paralives mod manifest formats.
- AppData\LocalLow CC / script-mod surface (deferred to a later iteration).
- Custom load-order UI.
- macOS support. Vortex is Windows-only; Paralives is Mac-native but that's
  not addressable from a Vortex extension.

## Approach

**Option A: pure GDL, scaffold from `gdl init`.** The whole extension
declared in `game.yaml`, with `src/hooks.ts` empty at v1. GDL compiles to a
webpack-bundled `dist/extension.js` plus an `info.json` and ships as a
zip via `gdl package`.

Rejected alternatives:

- **Hand-rolled TS extension** (Luma-Island shape, ~80–120 lines of JS):
  discards the value GDL offers, more code to maintain, defeats the stated
  goal of using GDL.
- **Hybrid: GDL submodule + thick `src/hooks.ts`** with proactive hooks for
  mod folder detection / Workshop dedup / manifest validation: premature
  code against an unknown API surface. YAGNI — easy to upgrade to this
  later by populating the existing `hooks.ts`.

## Repository layout

```
game-paralives/                          ← public GitHub repo (this directory)
├── game.yaml                            ← entire extension declaration
├── src/
│   └── hooks.ts                         ← empty stub at v1
├── package.json                         ← scripts delegate to gdl/dist/cli.js
├── vitest.config.ts                     ← runs .gdl-out/tests.gen.ts
├── tsconfig.json                        ← for hooks.ts type-checking
├── gameart.webp                         ← 512×288 logo (already in place)
├── README.md
├── LICENSE                              ← GPL-3.0
├── .gitignore                           ← node_modules/, dist/, .gdl-out/, out/
├── .gitmodules                          ← pins gdl/
├── gdl/                                 ← submodule → Nexus-Mods/game-description-language
└── .github/workflows/
    ├── ci.yml
    └── release.yml
```

Bootstrap with `node gdl/dist/cli.js init --game-id paralives --game-name "Paralives"`
then customize.

## `game.yaml`

```yaml
gdl: 1

game:
  id: paralives
  name: Paralives
  executable: Paralives.exe                  # TBD: verify post-EA
  requiredFiles:
    - Paralives.exe                          # TBD: verify post-EA
  logo: gameart.webp

stores:
  steam: 1118520                             # store.steampowered.com/app/1118520

modTypes:
  - id: paralives-overlay
    name: Paralives Overlay
    path: "${installPath}"

installers:
  - id: paralives-overlay
    priority: 90
    when: { hasFile: "**/*" }                # any archive matches
    anchor: "**/*"
    take: archive-root
    placeAt: "${installPath}"
    modType: paralives-overlay

nexus:
  modId: TBD                                 # populate once Nexus slug reserved
  fileGroupId: TBD
  displayName: "Paralives Support"
```

**Behavior.** Vortex auto-discovers Paralives via Steam App ID `1118520`.
Any archive a user installs is treated as a `paralives-overlay` mod; its
files hardlink into the game's install directory, preserving the archive's
own folder structure (`take: archive-root` means everything from the archive
root downward is laid out as-is). Conflicts between mods, undeploy/redeploy
on disable, and removal on uninstall are all handled by Vortex's standard
deployment engine.

**`take: archive-root` rationale.** No assumed manifest filename, so we
can't pick an anchor smarter than "the whole archive". If mod authors do
standardize on a manifest filename post-EA, we can add a higher-priority
installer that anchors on it and `take: parent` to strip wrapper folders.

## Hooks (`src/hooks.ts`)

```ts
export {};
```

Empty file at v1. Reserved for future use: `detectGameVersion` for
`discovery.version`, `regenerateModsTxt`-style `events.did-deploy` hooks if
Paralives needs a written load-order file, custom toolbar handlers.

## CI and distribution

**`package.json`** delegates to GDL:

```json
{
  "name": "game-paralives",
  "private": true,
  "scripts": {
    "build":   "node gdl/dist/cli.js build",
    "package": "node gdl/dist/cli.js package",
    "test":    "vitest run",
    "init-gdl": "cd gdl && pnpm install && pnpm build"
  }
}
```

**Submodule pin.** `gdl/` tracks a specific commit. Bumping is a deliberate
`git -C gdl pull && git add gdl` commit — no surprise GDL changes mid-release.

**CI — `.github/workflows/ci.yml`.** Prefer GDL's reusable workflow if one
is exposed at a stable path inside the submodule. Otherwise inline:
checkout with `submodules: recursive`, `pnpm install && pnpm build` in
`gdl/`, then `node gdl/dist/cli.js build && pnpm test` at the repo root.

**Release — `.github/workflows/release.yml`** fires on `v*` tags:

1. Checkout with `submodules: recursive`.
2. Build GDL.
3. `gdl package` → `out/paralives-vortex-v<ver>.zip`.
4. Attach zip to the GitHub release.
5. Upload to Nexus via `Nexus-Mods/upload-action` using `nexus.modId` /
   `nexus.fileGroupId` from `game.yaml`. **Gated on those values being
   non-`TBD`**; no-op until a Nexus slug is reserved.

**Secrets.** `NEXUS_API_KEY` in repo secrets (from
nexusmods.com/users/myaccount?tab=api) for the Nexus upload step.
GitHub release upload uses the default `GITHUB_TOKEN`.

**Versioning.** Manual semver via git tags. v0.x while EA-dependent values
(`Paralives.exe`, Nexus slug) are unverified. v1.0.0 once an end-to-end
install of a real mod works on a real Paralives install.

**GDL submodule access.** The GDL repo is currently private. Prerequisite:
either make GDL public before tagging the first release (simplest — CI
clones without secrets), or configure a deploy key / fine-grained PAT for
the workflow. Spec assumes the first will happen; otherwise the release
workflow has an additional secret to wire up.

## Tests

GDL codegens vitest specs from inline `tests:` cases in `game.yaml`. Seed
with archive-shape fixtures:

```yaml
tests:
  cases:
    - name: "loose file overlay drops to install root"
      files: ["MyMod/textures/foo.png", "MyMod/readme.txt"]
      expect:
        modType: paralives-overlay
        installer: paralives-overlay
        instructions:
          - { source: "MyMod/textures/foo.png", destination: "MyMod/textures/foo.png" }
          - { source: "MyMod/readme.txt",       destination: "MyMod/readme.txt" }
    - name: "nested archive root passes through unchanged"
      files: ["wrapper/MyMod/data.bin"]
      expect:
        modType: paralives-overlay
        instructions:
          - { source: "wrapper/MyMod/data.bin", destination: "wrapper/MyMod/data.bin" }
```

Exact `expect:` shape will match what GDL's `subnautica2-shaped` fixture
emits — we'll align with that during implementation rather than inventing
schema.

## Verification plan

**Pre-launch (before May 25):**

1. `pnpm install && pnpm build` inside `gdl/` succeeds.
2. `node gdl/dist/cli.js build` at repo root produces `dist/extension.js`
   and `.gdl-out/info.json` with no validator errors.
3. `pnpm test` passes the inline cases.
4. Sideload the built extension into a Vortex install on a Windows VM.
   Vortex should list "Paralives" without crashing — `requiredFiles`
   discovery fails (game not installed), which is expected.
5. Confirm GDL supports `${installPath}` in `modType.path` (Subnautica 2
   uses it, so this should be safe).

**Post-EA (May 25+):**

1. Install Paralives via Steam.
2. Confirm `Paralives.exe` is the executable; update `game.yaml` if not.
3. Confirm Vortex auto-discovers the install via Steam App ID `1118520`.
4. Build a trivial test mod (one text file in a zip), install via Vortex,
   deploy, confirm the file lands in the install root.
5. Tag `v0.1.0`, push, verify release workflow lands the zip on GitHub.
6. Coordinate with Nexus to reserve a Paralives game slug + a site mod
   listing for the extension. Fill in `nexus.modId` / `nexus.fileGroupId`.
   Re-release as `v0.1.1`.

## Risks and open questions

- **`${installPath}` in `modType.path`** — assumed supported by GDL based
  on Subnautica 2's usage. Verify during pre-launch step 5.
- **Executable name** — `Paralives.exe` is the natural guess; confirm
  post-EA. Wrong value means Vortex's `requiredFiles` check fails and
  discovery silently produces nothing.
- **GDL submodule access in CI** — see above; depends on GDL's
  public/private status at release time.
- **Nexus slug** — no Paralives game page exists on Nexus as of today.
  Needs coordination with Nexus staff. Release workflow's Nexus step is
  gated on a non-`TBD` slug.

## Future work (deferred, not v1)

- Separate mod types for AppData\LocalLow CC mods (`Mods/` subfolder once
  confirmed), with a manifest-aware installer.
- A dedicated mod type for community script mods (BepInEx-style, if a
  community script loader for Paralives emerges).
- Steam Workshop subscription detection / dedup.
- Manifest parsing for richer mod metadata in Vortex UI.
- Toolbar action "Open Paralives Mods folder" once a real folder exists.
- Custom load-order UI if feature mods turn out to need ordering.
