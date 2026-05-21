# Paralives Vortex Extension Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a v1 Vortex game extension for Paralives that auto-discovers a Steam install and deploys any archive's contents as a hardlink overlay on the game's install directory, declared almost entirely in YAML via `Nexus-Mods/game-description-language` (GDL).

**Architecture:** GDL as a git submodule at `gdl/`. A single `game.yaml` declares game discovery (Steam App ID `1118520`), one mod type (`paralives-overlay`, deployed to `${installPath}`), and one permissive installer that anchors on any file in the archive and lays it out at the archive root. `src/hooks.ts` is an empty `export {}` stub. CI builds via GDL's CLI; releases on `v*` tags package the extension and (when Nexus IDs are populated) upload to a Nexus site listing.

**Tech Stack:** Node 22+ (we have 25.9 via mise), pnpm, TypeScript, vitest, GDL CLI (`node gdl/dist/cli.js`), GitHub Actions, Vortex extension API (loaded at runtime as webpack external).

**Pre-existing state in this repo:**
- `gameart.webp` (512×288, 25KB) is in place at repo root, untracked.
- Design spec committed at `docs/superpowers/specs/2026-05-21-paralives-vortex-extension-design.md` (commit `2fb3632`).
- `main` branch initialized; no submodules; no `package.json` yet.

---

## Task 1: Add GDL as a git submodule and build it

**Files:**
- Create: `gdl/` (submodule)
- Create: `.gitmodules`

**Why TDD doesn't fit this task:** This is one-time tooling bootstrap. Verification is "does the CLI run." No behavior to test.

- [ ] **Step 1: Add the GDL submodule**

Run from repo root (`/home/tbaldrid/oss/game-paralives`):

```bash
git submodule add https://github.com/Nexus-Mods/game-description-language.git gdl
```

Expected: clones into `gdl/`, creates `.gitmodules`. If the GDL repo is still private and HTTPS prompts for credentials, use SSH instead: `git@github.com:Nexus-Mods/game-description-language.git`. Note in commit message which transport was used.

- [ ] **Step 2: Install GDL's dependencies and build it**

```bash
cd gdl && pnpm install && pnpm build && cd ..
```

Expected: `gdl/dist/cli.js` exists. `gdl/node_modules/` exists (gitignored by GDL's own `.gitignore`, not ours). If pnpm complains about Node version, confirm `node --version` is >= 22.

- [ ] **Step 3: Verify the CLI is callable**

```bash
node gdl/dist/cli.js --help
```

Expected: usage text listing subcommands including `build`, `package`, `init`, `test:corpus`, `publish-info`.

- [ ] **Step 4: Commit**

```bash
git add .gitmodules gdl
git commit -m "chore: add game-description-language as gdl/ submodule"
```

---

## Task 2: Scaffold project files with `gdl init`

**Files (created by `gdl init`):**
- Create: `game.yaml`
- Create: `src/hooks.ts`
- Create: `package.json`
- Create: `vitest.config.ts`
- Create: `tsconfig.json`
- Create: `.gitignore`
- Create: `README.md`
- Possibly: `gameart.jpg` placeholder (we'll remove in favor of existing `gameart.webp`)

**Why TDD doesn't fit:** Scaffolding step. We verify what `gdl init` produced and adapt.

- [ ] **Step 1: Run `gdl init`**

```bash
node gdl/dist/cli.js init --game-id paralives --game-name "Paralives"
```

Expected: emits the files listed above. If the command refuses to overwrite the existing `gameart.webp` or `docs/` directory, that's correct — we want it to leave those alone.

- [ ] **Step 2: Inspect what was generated**

```bash
ls -la
cat package.json
cat .gitignore
cat game.yaml
```

Confirm:
- `package.json` has scripts that invoke `gdl` (build, package, test).
- `.gitignore` lists `node_modules/`, `dist/`, `.gdl-out/`, `out/`.
- `game.yaml` has at minimum `gdl: 1`, a `game:` block with `id: paralives`, `name: Paralives`.
- `src/hooks.ts` exists.

- [ ] **Step 3: Remove any placeholder `gameart.jpg`**

If `gdl init` created a placeholder `gameart.jpg`:

```bash
rm -f gameart.jpg
```

Our `gameart.webp` (already on disk) is the real asset.

- [ ] **Step 4: Commit the scaffold**

```bash
git add -A
git commit -m "chore: scaffold extension with gdl init"
```

(Use `git add -A` here because `gdl init` produced many files in one shot; do not use `-A` in later tasks.)

---

## Task 3: Configure `package.json` and confirm `pnpm install` succeeds

**Files:**
- Modify: `package.json`

**Why TDD doesn't fit:** Build config. Verification is "does `pnpm install` and `pnpm build` both work."

- [ ] **Step 1: Ensure `package.json` matches the spec**

Open `package.json`. Confirm or set the following fields. If `gdl init` already produced something close, edit minimally rather than rewriting wholesale.

```json
{
  "name": "game-paralives",
  "version": "0.0.1",
  "private": true,
  "description": "Vortex extension for Paralives",
  "scripts": {
    "build":   "node gdl/dist/cli.js build",
    "package": "node gdl/dist/cli.js package",
    "test":    "vitest run",
    "init-gdl": "cd gdl && pnpm install && pnpm build"
  }
}
```

Keep any `devDependencies` (vitest, typescript) that `gdl init` added — don't strip them.

- [ ] **Step 2: Install dependencies at the repo root**

```bash
pnpm install
```

Expected: produces a `pnpm-lock.yaml` and `node_modules/`. No errors.

- [ ] **Step 3: Confirm `pnpm build` runs (even if YAML is still skeletal)**

```bash
pnpm build
```

Expected outcomes (either is fine at this stage):
- **Success:** `dist/extension.js` and `.gdl-out/` produced.
- **Validator failure** with a clear error pointing at missing `modTypes` / `installers` in `game.yaml`. We fix this in Task 5.

If it fails for any *other* reason (cannot find `gdl/dist/cli.js`, Node version, etc.), stop and resolve before proceeding.

- [ ] **Step 4: Commit**

```bash
git add package.json pnpm-lock.yaml
git commit -m "chore: configure package.json scripts and install deps"
```

(`node_modules/` should be gitignored — confirm `git status` shows it untracked.)

---

## Task 4: Replace placeholder asset reference and add the bare game block

**Files:**
- Modify: `game.yaml`

**Why TDD partly fits:** The build serves as the test. We modify `game.yaml`, then `pnpm build` must succeed (or produce only the expected "no installers" error).

- [ ] **Step 1: Set the full `game:` block, `stores:`, and `logo:`**

In `game.yaml`, ensure the `game:` and `stores:` blocks are exactly:

```yaml
gdl: 1

game:
  id: paralives
  name: Paralives
  executable: Paralives.exe
  requiredFiles:
    - Paralives.exe
  logo: gameart.webp

stores:
  steam: 1118520
```

(Whatever `gdl init` generated for these fields gets overwritten with the above.)

- [ ] **Step 2: Run the build and check the error**

```bash
pnpm build
```

Expected: the validator should now complain about the missing `modTypes:` and/or `installers:` blocks but be happy with everything we wrote. Read the error carefully; if it mentions `executable`, `requiredFiles`, `stores`, or `logo`, those need fixing in this task. If it mentions `modTypes` or `installers`, that's expected and fixed in Task 6.

- [ ] **Step 3: Commit**

```bash
git add game.yaml
git commit -m "feat: declare Paralives game identity and Steam discovery"
```

---

## Task 5: Write the failing inline test cases

**Files:**
- Modify: `game.yaml`

**Why TDD fits:** We define expected installer behavior *before* implementing the installer. Tests must fail (or build must fail) before Task 6 makes them pass.

- [ ] **Step 1: Look up the exact `expect:` schema from the subnautica2 fixture**

Read GDL's reference fixture to copy the schema shape:

```bash
cat gdl/tests/fixtures/subnautica2-shaped/game.yaml | grep -A 40 '^tests:'
```

Identify what fields each test case under `tests.cases[]` uses (`name`, `files`, `expect.modType`, `expect.installer`, `expect.instructions[].source`, `expect.instructions[].destination`). If the fixture uses a slightly different key name (e.g. `outputs` instead of `instructions`), match the fixture exactly.

- [ ] **Step 2: Append a `tests:` block to `game.yaml`**

Append to `game.yaml` (adjusting field names per Step 1's findings if they differ):

```yaml
tests:
  cases:
    - name: "loose file overlay drops to install root"
      files:
        - "MyMod/textures/foo.png"
        - "MyMod/readme.txt"
      expect:
        modType: paralives-overlay
        installer: paralives-overlay
        instructions:
          - { source: "MyMod/textures/foo.png", destination: "MyMod/textures/foo.png" }
          - { source: "MyMod/readme.txt",       destination: "MyMod/readme.txt" }

    - name: "nested archive root passes through unchanged"
      files:
        - "wrapper/MyMod/data.bin"
      expect:
        modType: paralives-overlay
        installer: paralives-overlay
        instructions:
          - { source: "wrapper/MyMod/data.bin", destination: "wrapper/MyMod/data.bin" }
```

- [ ] **Step 3: Run the build — expect failure**

```bash
pnpm build
```

Expected: validator failure — either "no modType named `paralives-overlay`" or "no installers defined." Either is the correct failing state. If `gdl build` somehow succeeds anyway, run `pnpm test` and confirm that vitest fails with no tests to run or with test failures. We need a red state before Task 6.

- [ ] **Step 4: Commit the failing tests**

```bash
git add game.yaml
git commit -m "test: declare expected overlay installer behavior (failing)"
```

---

## Task 6: Add modType and installer to make tests pass

**Files:**
- Modify: `game.yaml`

**Why TDD fits:** This is the minimal implementation that turns the previous task's red into green.

- [ ] **Step 1: Add `modTypes:` and `installers:` blocks**

Insert between `stores:` and `tests:` in `game.yaml`:

```yaml
modTypes:
  - id: paralives-overlay
    name: Paralives Overlay
    path: "${installPath}"

installers:
  - id: paralives-overlay
    priority: 90
    when: { hasFile: "**/*" }
    anchor: "**/*"
    take: archive-root
    placeAt: "${installPath}"
    modType: paralives-overlay
```

- [ ] **Step 2: Run the build**

```bash
pnpm build
```

Expected: success. Produces `dist/extension.js`, `.gdl-out/extension.ts`, `.gdl-out/tests.gen.ts`, `.gdl-out/info.json`. No validator errors.

- [ ] **Step 3: Run the tests**

```bash
pnpm test
```

Expected: both test cases pass. If a test fails because the `expect:` schema we wrote in Task 5 doesn't match what GDL's test codegen emits, fix the schema in `game.yaml` and re-run. Do not weaken the assertion to make it pass — fix the field names to match what GDL actually checks.

- [ ] **Step 4: Inspect `.gdl-out/extension.ts` and confirm `${installPath}` interpolation**

```bash
grep -n 'installPath\|getPath' .gdl-out/extension.ts
```

Expected: a `getPath` (or equivalent) function uses Vortex's discovered install path, *not* a literal `"${installPath}"` string. This validates the spec's "Risk #1" — that GDL supports `${installPath}` in `modType.path`. If we see the literal string in the output, that's a GDL bug to file (or patch, since we have admin on GDL); flag and stop.

- [ ] **Step 5: Commit**

```bash
git add game.yaml
git commit -m "feat: paralives-overlay mod type and overlay installer"
```

---

## Task 7: Ensure `src/hooks.ts` is the empty stub

**Files:**
- Modify (or confirm): `src/hooks.ts`

**Why TDD doesn't fit:** No behavior. Just confirming the file is the empty stub the spec calls for.

- [ ] **Step 1: Overwrite `src/hooks.ts` to be exactly:**

```ts
// src/hooks.ts
// No hooks needed at v1. File exists so the GDL submodule and TS config
// have something to point at, and post-EA additions land in a known place.
export {};
```

- [ ] **Step 2: Rebuild to confirm nothing breaks**

```bash
pnpm build && pnpm test
```

Expected: both still pass.

- [ ] **Step 3: Commit (only if there was a change)**

```bash
git add src/hooks.ts
git status
git diff --cached --quiet || git commit -m "chore: keep src/hooks.ts as empty stub"
```

If `git diff --cached --quiet` succeeds (no staged changes), skip the commit — `gdl init` already produced the right content.

---

## Task 8: Write `.github/workflows/ci.yml`

**Files:**
- Create: `.github/workflows/ci.yml`

**Why TDD doesn't fit:** GitHub Actions config; verification happens when you push.

- [ ] **Step 1: Create the workflow**

Write to `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout (with submodules)
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Build GDL submodule
        working-directory: gdl
        run: |
          pnpm install --frozen-lockfile
          pnpm build

      - name: Install extension deps
        run: pnpm install --frozen-lockfile

      - name: Build extension
        run: pnpm build

      - name: Run tests
        run: pnpm test
```

- [ ] **Step 2: Verify locally that the same sequence works**

From a clean shell (no cached state), run the commands the workflow runs:

```bash
(cd gdl && pnpm install --frozen-lockfile && pnpm build) \
  && pnpm install --frozen-lockfile \
  && pnpm build \
  && pnpm test
```

Expected: all green.

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: build and test on push and PR"
```

---

## Task 9: Write `.github/workflows/release.yml`

**Files:**
- Create: `.github/workflows/release.yml`

**Why TDD doesn't fit:** Release infra; verification is "does it run when we tag."

- [ ] **Step 1: Create the workflow**

Write to `.github/workflows/release.yml`:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout (with submodules)
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 9

      - name: Build GDL
        working-directory: gdl
        run: |
          pnpm install --frozen-lockfile
          pnpm build

      - name: Install extension deps
        run: pnpm install --frozen-lockfile

      - name: Package extension
        run: pnpm package

      - name: Read Nexus IDs from game.yaml
        id: nexus
        run: |
          MOD_ID=$(node gdl/dist/cli.js publish-info nexus.modId 2>/dev/null || echo TBD)
          FG_ID=$(node gdl/dist/cli.js publish-info nexus.fileGroupId 2>/dev/null || echo TBD)
          echo "modId=$MOD_ID"       >> "$GITHUB_OUTPUT"
          echo "fileGroupId=$FG_ID"  >> "$GITHUB_OUTPUT"

      - name: Upload zip to GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: out/*.zip

      - name: Upload to Nexus Mods
        if: steps.nexus.outputs.modId != 'TBD' && steps.nexus.outputs.fileGroupId != 'TBD'
        uses: Nexus-Mods/upload-action@v1
        with:
          game-id: site
          mod-id:        ${{ steps.nexus.outputs.modId }}
          file-group-id: ${{ steps.nexus.outputs.fileGroupId }}
          file:          out/*.zip
          api-key:       ${{ secrets.NEXUS_API_KEY }}
```

Notes for the implementer:
- The `publish-info` subcommand exists in GDL per the research; double-check the exact CLI shape with `node gdl/dist/cli.js publish-info --help` and adjust if needed.
- The Nexus upload step is the official `Nexus-Mods/upload-action` per the research. Pin the version once you confirm the latest tag.
- `game-id: site` is the Nexus convention for extension uploads (extensions live under the `site` domain on Nexus, like Luma Island's).

- [ ] **Step 2: Verify the YAML is syntactically valid**

```bash
python3 -c 'import sys, yaml; yaml.safe_load(open(".github/workflows/release.yml"))' && echo OK
```

Expected: prints `OK`. (If `python3` isn't available, skip — GitHub will validate on push.)

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/release.yml
git commit -m "ci: release workflow on v* tags, Nexus upload gated on slug"
```

---

## Task 10: Add LICENSE and README

**Files:**
- Create: `LICENSE`
- Modify or create: `README.md`

**Why TDD doesn't fit:** Docs/legal.

- [ ] **Step 1: Add the GPL-3.0 LICENSE**

Fetch the canonical GPL-3.0 text and place it at `LICENSE`. Easiest source:

```bash
curl -L https://www.gnu.org/licenses/gpl-3.0.txt -o LICENSE
```

Verify the file starts with `GNU GENERAL PUBLIC LICENSE` and is ~35KB.

- [ ] **Step 2: Write the README**

Overwrite (or create) `README.md` with:

```markdown
# game-paralives — Vortex extension for Paralives

Adds [Paralives](https://store.steampowered.com/app/1118520/) support to the
[Vortex](https://www.nexusmods.com/about/vortex/) mod manager.

**Status:** v0.x — early. Paralives Early Access ships May 25, 2026, and a
number of values in `game.yaml` (executable name, mod folder layout) are
verified only after EA is available. Expect rapid iteration.

## What it does (v1)

- Auto-discovers a Steam install of Paralives (App ID `1118520`).
- Treats any installed archive as a "Paralives Overlay" mod and hardlinks
  its contents into the game's install directory. Vortex handles conflict
  resolution and undeploy as normal.

## Install

**End users:** drag the latest release `.zip` from [Releases](../../releases)
onto Vortex's Extensions tab, restart Vortex.

**From source:**

```bash
git clone --recursive https://github.com/<you>/game-paralives.git
cd game-paralives
pnpm install
pnpm build
pnpm package
# out/paralives-vortex-v<ver>.zip — drag into Vortex
```

## Develop

```bash
pnpm install              # one time
pnpm build                # produces dist/extension.js
pnpm test                 # runs vitest against codegen'd test cases
pnpm package              # produces out/*.zip
```

The whole extension is declared in `game.yaml`. See
[`docs/superpowers/specs/`](docs/superpowers/specs/) for the design.

## License

GPL-3.0. See [LICENSE](LICENSE).
```

- [ ] **Step 3: Commit**

```bash
git add LICENSE README.md
git commit -m "docs: README and GPL-3.0 LICENSE"
```

---

## Task 11: Commit the game art and final smoke check

**Files:**
- Stage: `gameart.webp` (already on disk, untracked)

- [ ] **Step 1: Stage and commit `gameart.webp`**

```bash
git add gameart.webp
git commit -m "chore: add game art (512x288 webp)"
```

- [ ] **Step 2: Final clean-build smoke**

```bash
rm -rf node_modules dist .gdl-out out
(cd gdl && pnpm install --frozen-lockfile && pnpm build)
pnpm install --frozen-lockfile
pnpm build
pnpm test
pnpm package
ls -lh out/
```

Expected:
- `pnpm build` succeeds.
- `pnpm test` shows both cases pass.
- `out/` contains `paralives-vortex-v0.0.1.zip` (or similar — name derives from `package.json`'s `version` and `game.yaml`'s `game.id`).

- [ ] **Step 3: Sanity-check the zip contents**

```bash
unzip -l out/*.zip
```

Expected contents:
- `info.json`
- `extension.js` (the webpack bundle)
- `gameart.webp`

If any of those is missing, fix before claiming v1 done.

- [ ] **Step 4: Commit any incidental changes from the smoke (probably none)**

```bash
git status
```

Expected: clean tree. If something changed (e.g. lockfile drift), inspect and commit.

---

## Out of scope for this plan (carried from the spec)

These do **not** appear as tasks here. They are post-implementation activities:

- Pushing the repo to GitHub and configuring branch protection.
- Setting the `NEXUS_API_KEY` repository secret.
- Tagging `v0.1.0` and verifying the release workflow runs end-to-end.
- Reserving a Nexus slug for Paralives and a site mod listing for the
  extension; populating `nexus.modId` / `nexus.fileGroupId` and re-tagging.
- Installing Paralives on May 25 and verifying `Paralives.exe`, install
  discovery, and a real mod overlay deploy.
- Sideload-testing the extension in a Vortex install on Windows.

These should be tracked separately (issues / a follow-up plan) once the EA
build is available.
