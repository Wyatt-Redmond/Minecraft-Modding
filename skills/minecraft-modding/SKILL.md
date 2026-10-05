---
name: minecraft-modding
description: >-
  Port, write, or debug Minecraft mods — especially across a game-version or
  mod-loader change (Forge/NeoForge → Fabric, old Fabric forward, 1.21.x → 26.2),
  or building a new multiloader mod from scratch. Use when the task involves
  porting a mod to a new MC version or loader, setting up Gradle/Loom for a mod,
  the Architectury common/fabric/forge/neoforge layout, cross-loader registration
  / networking / menus, the retained-mode GUI rewrite (GuiGraphics → extract),
  the in-world submit render pipeline, GeckoLib, mixin crash guards, or diagnosing
  modding failures like a black screen, purple/invisible blocks or items, missing
  icons, "Components not bound yet", "Item id not set", mixin APPLY failures, or
  modpack distribution via CurseForge. Triggers on mentions of: Fabric, Forge,
  NeoForge, Loom, mojmap, mixin, Architectury, GeckoLib, Patchouli, modpack,
  CurseForge, Sinytra Connector, or a Minecraft version like 1.20.1 / 1.21 / 26.2 / 26.3.
---

# Minecraft modding

Porting an existing mod, writing a new one, bumping a mod to a newer MC version,
or debugging mod/modpack failures — for a game-version jump, a loader change, or a
greenfield multiloader build. The worked technical detail targets the **1.21.x
Forge/NeoForge → 26.2 Fabric** delta, with a second worked example for the
**26.2 → 26.3** bump; the methodology is version-agnostic. For any version you
don't have a cheatsheet for, **don't guess the delta — derive it with `javap`
against the target MC jar** (the repeatable method is in
[version-bump-method.md](references/version-bump-method.md)).

## Era map — orient before you start

This skill is tuned for **2026+ Minecraft — the 26.x line and forward** (26.2 and 26.3 are the
worked examples); treat that content as current and accurate, and derive any later 2026+ version's
delta with the `javap` method. For **pre-2026 (1.x) eras it gives orientation, not depth** — enough
to place your target on the map below and avoid the modern-era assumptions that would mislead there,
then **derive the exact API delta with the `javap` method** in
[version-bump-method.md](references/version-bump-method.md). Locate your target's era first — it
fixes the toolchain, mapping system, and which of this skill's specifics apply.

| MC range | Loader(s) + toolchain | Mapping (dev) | Era-defining traits |
|---|---|---|---|
| **≤ 1.12.2** | Forge/FML · ForgeGradle | MCP/SRG (obf at runtime) | pre-"flattening" metadata ids; **no standard Mixin** (ASM coremods instead); no datapacks |
| **1.13 – 1.16** | Forge (ForgeGradle) · Fabric + Loom (yarn/intermediary → `remapJar`) | SRG / yarn | 1.13 flattening; datapacks arrive (1.13); Fabric emerges |
| **1.17** | + Java 16 | SRG / yarn | `net.minecraft` package rename; JDK bump |
| **1.18 – 1.19** | Forge · Fabric · *(NeoForge forks at 1.20.2)* | SRG (official names in dev ~1.16.5+, SRG at runtime) / yarn | 1.18 datapack-driven worldgen; 1.19 registry rework |
| **1.20 – 1.20.4** | Forge/NeoForge · Fabric | mojmap (NeoForge) / yarn | `GuiGraphics` arrives (1.20); items still NBT |
| **1.20.5 – 26.3** | Fabric mojmap no-remap · NeoForge **ModDevGradle** (NeoGradle legacy) · Forge ForgeGradle | mojmap | data components (1.20.5); `Properties.setId` (1.21.2); retained-mode GUI (26.2); Feature-interface rewrite (26.3) |

The table only orients you to the right toolchain/mapping — the API specifics for *your* version
still come from `javap` against that version's jar, never from copying another era's cheatsheet.

For the **exact per-version numbers across the 2026+ line (26.1 → 26.3)** — Java, resource/data pack
formats, mappings, Fabric loader/API, NeoForge, Forge, Loom, GeckoLib, LWJGL — use the baseline
matrix in [version-baselines.md](references/version-baselines.md) (verified against live sources; it
also records the "verify before you bump" endpoints and the diverged resource-vs-data pack formats).

## First move: which job is this?

1. **Porting an existing mod** → start with the **decision tree below**, triage
   *before* writing code, then read [porting-playbook.md](references/porting-playbook.md).
2. **Bumping a mod to a newer MC version** (same loader, e.g. 26.2→26.3) → read
   [version-bump-method.md](references/version-bump-method.md): the `javap`-guided compile loop to
   derive *any* version's delta, plus the full 26.2→26.3 delta as a worked example.
3. **Writing a new mod from scratch** → read [writing-from-scratch.md](references/writing-from-scratch.md)
   (clean multiloader defaults so you never accrue the port debt).
4. **Debugging a specific failure** → jump to the matching section:
   crash guards & black-screen/purple-texture fixes live in
   [porting-playbook.md](references/porting-playbook.md) §6 (crash-guard table) and §8
   (failure-mode table); the pip-preview bug is [useful-ribbits-case-study.md](references/useful-ribbits-case-study.md) §4.

Do not source-port a mod that doesn't need it — most never reach rung 6.

## Port decision tree (run per mod, before any code)

```
├─1. Official build for TARGET version + loader exists?      → USE IT. Done.
├─2. Official build on the OTHER loader?                     → compat layer (all-or-nothing) or treat as "no build"
├─3. Mod is a BACKPORT of a now-vanilla feature?             → DELETE it (verify feature is in the target jar)
├─4. Native equivalent / refork for your loader?             → SWAP it (cheaper, already maintained)
├─5. Repo has a multiloader / newer-version branch?          → LOADER-GLUE / small bump, not a full lift (cheap)
├─6. Published SOURCE (any version) exists?                  → SOURCE-PORT (cost scales with version delta + GUI/render surface)
└─7. No source, no build, no equivalent, not vanilla?        → DROP it, and tell the user exactly what is lost
```

Most mods exit at 1–5. Spend the budget on 6–7. Full tree + the "what makes a
mod hard" tiering: [porting-playbook.md](references/porting-playbook.md) §1 and §10.

## Non-negotiables (the traps that bite every time)

- **Settle the mapping/runtime model first** — it dictates the whole Gradle/Loom
  config. 26.2 Fabric ships **mojmap, no remap**; older Fabric ships intermediary
  and needs `remapJar`. Verify by grepping *extracted* classes for `class_####`/`method_####`.
  (That grep is a **Fabric-only** test. Forge *dev* uses ForgeGradle's `official` channel — Mojang
  names, no param names, since 1.14+ — with SRG `func_`/`field_` applied only at *runtime* by `reobf`;
  NeoForge *dev* is mojmap via ModDevGradle. Same settle-first rule, different detector.)
- **Clean compile only.** `clean compileJava` every time — javac short-circuits,
  so a shrinking error count is a lie. Measure by emitted `.class` count or a green compile.
- **An object's id must be on its `Properties` at construction** (`setId(key)`), or
  the ctor throws `Item id not set`. Prime codemod target. *(1.21.2+ only — older versions
  take the id from the `Registry.register` call (Fabric 1.14+) or `setRegistryName`
  (Forge ≤1.12.2), not from `Properties`; `setId` won't exist there.)*
- **Never build an `ItemStack` / call `getDefaultInstance()` in a `<clinit>`** of
  any class that loads during resource reload (BER, renderer, model, particle) →
  `Components not bound yet`. Make it a lazy `Supplier`. *(The crash is 1.20.5+ data-component
  era; pre-1.20.5 items are NBT and can't throw it — but the lazy-`Supplier` discipline for
  registry-touching statics still applies everywhere.)*
- **Centralize crash guards into one `"*"`-env mixin mod** (not `"client"`, or the
  dedicated server skips them), not per-content-jar edits. *(`"*"`/`"client"` is the Fabric
  `fabric.mod.json` env key; on Forge/NeoForge use dist checks + the mixin-json `client` array.
  Standalone-mixin guards assume a mixin-capable loader — Fabric, or Forge/NeoForge via a mixin
  coremod; pre-1.12.2 Forge predates standard Mixin and needs an ASM coremod / Forge hooks instead.)*
- **Edit jars only with the game CLOSED** — a live rewrite = `ZipException: bad LOC`,
  a fake "crash".
- **A shared-`common` change means rebuild EVERY loader**, not just the one you tested.
- **Datapack/JSON format changes don't fail the build** — they surface at *datapack load* (server
  start / "preparing for world creation"), one registry **in waves**. Don't drip-fix one per launch:
  on the first datapack error, audit the WHOLE pack against the target version's vanilla data and fix
  every file of each type at once (e.g. 26.3 renamed BlockState `"Name"→"id"`, loot discriminators
  `"condition"/"function"→"type"`, advancement `"recipe"→"recipes"`, dropped `_state_provider`).
  *(Era gate: datapacks exist 1.13+, datapack-driven worldgen 1.18+, JSON recipes 1.12+ — below
  those, this whole load-time-waves class doesn't exist and format errors surface at build/registration.)*
- **The build toolchain can lag a brand-new MC version** independently of your code — e.g. Architectury
  Loom's ASM throwing `Unsupported class file major version N` on a Java-N dep (26.3: LWJGL 3.4.3's
  multi-release classes). Code compiling ≠ jar packaging; see version-bump-method.md's toolchain gotcha.

## Validation stack — run in this order, skip nothing

Each stage is blind to what the previous one catches:

1. **Clean compile** (proves nothing about mixins/runtime)
2. **Boot-verify on a dedicated server** to `Done` — authoritative common-mixin check;
   grep the log for `FAILED during APPLY | was not applied` after *every* fix (faults surface in waves)
3. **Static mixin audit** (name-level only; misses `@At` descriptor mismatches)
4. **Client mixins are a server-boot BLIND SPOT** — validate each against the merged jar, or test on a real client
5. **Bake gate** (resolves blockstate → model → parent → textures; the objective purple/invisible check)
6. **Creative-inventory icon test** (26.2 needs a separate `assets/<ns>/items/<name>.json` def)
7. **RCON phantom scan** — `execute if block <pos> <id>` throws `Unknown block type` for unregistered ids
8. **In-client QA** — the last mile; boot-to-Done never exercises rendering

Three "it works" illusions live here (green compile, boot-to-Done, asset-chain
verify) — each necessary, none sufficient. Detail: [porting-playbook.md](references/porting-playbook.md) §7.

## References

| File | Read it for |
|---|---|
| [porting-playbook.md](references/porting-playbook.md) | The full method: decision tree, toolchain, cross-loader map, API-change cheatsheet + codemods, GUI/render cookbook, crash-guard library, validation stack, failure modes, distribution, difficulty tiering. |
| [version-bump-method.md](references/version-bump-method.md) | Bumping to a newer MC version (same loader): the `javap`-guided compile loop to derive any version's delta, the full 26.2→26.3 delta (code + datapack + toolchain), and how datapack errors surface in waves at world-load. |
| [version-baselines.md](references/version-baselines.md) | The 2026+ baseline matrix (26.1 → 26.3): per-version Java, resource/data pack formats, mappings, Fabric loader/API, NeoForge, Forge, Loom, GeckoLib, LWJGL — plus the mojmap/no-remap + shadowJar + LWJGL-pin facts true for all 26.x, and the verify-from-source endpoints. The "what Minecraft expects for 26.x+" lookup. |
| [official-docs-and-primers.md](references/official-docs-and-primers.md) | Official sources + the canonical per-loader toolchain (Fabric Loom · Forge ForgeGradle/MDK · NeoForge ModDevGradle), dev-mappings/metadata/entrypoint/registration per loader, Yarn-EOL migration tooling, the Java ladder, and the **NeoForge primer chain** (1.21.11→26.1→26.2→26.3) — the authoritative cross-loader source for the semantic vanilla deltas `javap` can't reveal. |
| [writing-from-scratch.md](references/writing-from-scratch.md) | Greenfield multiloader defaults: project layout, the platform seam, modern registration, data-components-over-NBT, JSON gotchas, retained-mode rendering, the "it builds and loads" checklist. |
| [useful-ribbits-case-study.md](references/useful-ribbits-case-study.md) | A real blow-by-blow port (1.20.1 Forge → 26.2 F/F/NF), including the chronological fix log and the deep-dive on the Forge pip-preview collapse. |

**Rule:** triage before you port, settle the runtime model before you build,
measure only with clean compiles, and treat green-compile / boot-to-Done / in-client
as three separate gates. The reusable takeaways are each section's closing **Rule**
in the playbook.
