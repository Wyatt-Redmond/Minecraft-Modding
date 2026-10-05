# Minecraft Modding — a Claude Code skill

A packaged [Claude Code](https://code.claude.com) skill for **porting, writing, and debugging
Minecraft mods** across game-version and mod-loader changes — Fabric, Forge, and NeoForge, from one
Architectury codebase. It is **tuned for 2026+ Minecraft (the 26.x line and forward)** with accurate,
verified baselines, and gives decent orientation for older (1.x) eras.

Once installed, Claude invokes it automatically when a task involves modding (or you can call it with
`/minecraft-modding`).

## What it covers

- **Porting** an existing mod to a new version/loader — a decision tree (most mods never need a source
  port), cross-loader conversion maps, codemod discipline, a crash-guard pattern library, and a
  layered validation stack.
- **Bumping** a mod to a newer MC version — a repeatable `javap`-guided method to derive *any*
  version's API delta yourself, with 26.2→26.3 as a worked example.
- **Writing** a new multiloader mod from scratch — clean defaults so you never accrue port debt.
- **Debugging** modding failures — black screens, purple/invisible blocks, missing icons,
  "Components not bound yet", mixin APPLY failures, and more.
- **26.x baselines & official sources** — a verified per-version matrix (Java, pack formats, mappings,
  loader/API/toolchain versions) and the authoritative docs + the NeoForge primer chain for the
  semantic vanilla deltas `javap` can't reveal.

## Requirements

None beyond Claude Code. It's pure documentation — no code to run, no dependencies.

## Install

### Option A — Plugin (one-command install)

From a Claude Code session:

```
/plugin marketplace add Wyatt-Redmond/minecraft-modding
/plugin install minecraft-modding@mc-skills
```

Or from the terminal:

```bash
claude plugin marketplace add Wyatt-Redmond/minecraft-modding
claude plugin install minecraft-modding@mc-skills
```

### Option B — Manual (copy the folder)

Copy the skill into your Claude skills directory:

```bash
git clone https://github.com/Wyatt-Redmond/minecraft-modding.git
cp -r minecraft-modding/skills/minecraft-modding ~/.claude/skills/
```

- `~/.claude/skills/minecraft-modding/` → personal (all your projects)
- `<project>/.claude/skills/minecraft-modding/` → project-scoped (commit it with a repo)

Claude Code auto-discovers the skill from `SKILL.md` on the next session — no registration needed.

## Layout

```
.
├── README.md
├── .claude-plugin/
│   ├── marketplace.json            # makes this repo a plugin marketplace
│   └── plugin.json                 # the plugin (lives at the repo root)
└── skills/
    └── minecraft-modding/          # the skill (copy THIS for manual install)
        ├── SKILL.md
        └── references/
```

## License

Licensed under the **MIT License** — see [`LICENSE`](LICENSE). © 2026 Wyatt Redmond.
