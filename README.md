# game-paralives — Vortex extension for Paralives

Adds [Paralives](https://store.steampowered.com/app/1118520/) support to the
[Vortex](https://www.nexusmods.com/about/vortex/) mod manager.

**Status:** v0.x — early. Paralives Early Access ships May 25, 2026, and a
number of values in `game.yaml` (executable name, mod folder layout, Nexus
mod-page IDs) are verified only after EA is available. Expect rapid iteration.

## What it does (v1)

- Auto-discovers a Steam install of Paralives (App ID `1118520`).
- Treats any installed archive as a "Paralives Overlay" mod and hardlinks its
  contents into the game's install directory. Vortex handles conflict
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
pnpm build                # produces dist/extension.js via gdl
pnpm test                 # builds and runs the codegen'd tests
pnpm package              # produces out/paralives-vortex-v<ver>.zip
```

The whole extension is declared in `game.yaml`. The build pipeline is
[`Nexus-Mods/game-description-language`](https://github.com/Nexus-Mods/game-description-language)
(loaded via the `gdl/` git submodule).

See [`docs/superpowers/specs/`](docs/superpowers/specs/) for the design and
[`docs/superpowers/plans/`](docs/superpowers/plans/) for the implementation plan.

## License

GPL-3.0. See [LICENSE](LICENSE).
