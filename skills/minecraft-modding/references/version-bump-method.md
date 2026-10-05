# Version bumps: the method + a worked example (26.2 → 26.3)

The porting-playbook's API cheatsheet (§4) is a snapshot of the **1.21.x → 26.2** delta. For any
*other* jump — 26.2→26.3, 26.3→26.4, and so on — there is no pre-made table. This file gives the
repeatable **method** for finding a version's delta yourself, then the real **26.2 → 26.3** delta as a
worked example (and proof the method works). Reuse the method; the specifics are era-dependent.

## The method — a `javap`-guided compile loop (not a rewrite)

A same-era bump (same loader, one MC version up) is mostly mechanical — but some single-version
steps are large (a flattening like 1.13, the `net.minecraft` package rename like 1.17, a registry
rework like 1.19). Size the job by the `javap` diff first; don't assume the next delta is as small
as 26.2→26.3.

1. **Copy the project** (preserve the working version — if it isn't a git repo, copy the dir). Bump
   *every* version in `gradle.properties`: MC, loader, Fabric API, GeckoLib, Forge, NeoForge, the
   `>=X <Y` ranges, `compatible_versions`. **Verify each artifact exists before trusting it** — query
   the then-current mavens (confirm each endpoint — hosts move): `meta.fabricmc.net/v2/versions/game`, the GeckoLib cloudsmith, and
   `.../maven-metadata.xml` for fabric-api / neoforge / forge / cloth / modmenu.
2. **Compile `common` first** (`./gradlew :common:compileJava`, clean). ⚠ The **first compile masks
   errors** — javac short-circuits per file, so unresolved imports hide body errors and the count
   *grows* as you fix. Fix a wave, recompile, repeat until green; never trust one pass.
3. For each break, **look up the new API with `javap` against the target MC jar**
   (`~/.gradle/caches/fabric-loom/<ver>/minecraft-merged.jar`, or wherever your toolchain caches
   the merged jar — the path varies by Loom version and loader; Forge/NeoForge use the
   ForgeGradle/NeoGradle decomp, not a Loom cache):
   - ⚠ `javap` reads whatever names your dev jar is mapped to — Mojang names in the mojmap era,
     yarn/intermediary on older Fabric, MCP/SRG on old Forge. If names come out as `class_1234` /
     `func_#####`, point `javap` at the *named* dev jar (or translate through mappings) instead.
   - `javap -p <fqcn>` — current signatures (spot the added/removed param, e.g. a new enum arg).
   - `javap -p -c <fqcn>` — how a vanilla caller builds it (copy the pattern).
   - `unzip -l <jar> | grep -i <name>` — find a renamed/moved class when "cannot find symbol".
   - When a whole class/package is gone, `javap` a vanilla sibling to learn the replacement shape.
4. **Then the loader modules** — platform helpers and client registration surface a few more
   loader-specific breaks (NeoForge often compiles when Fabric/Forge don't, and vice-versa).
5. **Datapack/JSON changes do NOT fail the build** — they surface at **datapack load** (server start
   or "preparing for world creation"), and in **waves** (one registry at a time). Don't drip-fix one
   per launch: once you hit the first datapack error, **audit the whole datapack against vanilla** of
   the target version (`unzip` sample files from the MC jar, compare, fix every file of that type at
   once), then move to the next registry type.
6. **Defer cosmetic reworks** (a first-person render mixin, a minor convenience) behind a documented
   TODO rather than blocking a testable build.
7. **The toolchain can lag a brand-new version independently of your code** — see the gotcha at the end.

## The 26.2 → 26.3 delta (what the method found)

### Worldgen Feature system — rewritten
- `Feature` is now an **interface**; `FeatureConfiguration` and every `*Configuration` record are gone,
  and **`ConfiguredFeature` was removed entirely**. A feature now **carries its own config fields** and
  is **codec-dispatched**: implement `MapCodec<? extends Feature> codec()` +
  `boolean place(WorldGenLevel, ChunkGenerator, RandomSource, BlockPos)` (no `FeaturePlaceContext`).
  Merge your old `Feature<FC>` + its `FC` into one record/class.
- Register the feature's **`CODEC` into `FEATURE_TYPE`** (`Registry<MapCodec<? extends Feature>>`) —
  same shape as the `STRUCTURE_PROCESSOR` registry. The old `Feature<?>`-instance registry is gone.
- `PlacedFeature` now holds `Holder<Feature>` directly.

### Blocks
- **The per-block MapCodec / `codec()` system was removed** — no block declares `CODEC`/`codec()` or
  calls `propertiesCodec()` anymore. Delete them.
- `BonemealableBlock` gained a **`BonemealSource`** param on `isValidBonemealTarget` /
  `isBonemealSuccess` / `performBonemeal` (`BonemealSource.MOB` from a mob, `INTERACTION` from a player).
- `PushReaction.DESTROY` → **`POPPED`**; the whole enum was renamed (new values: `PUSH_PULL`, `PUSH`,
  `POPPED`, `IMMOVEABLE`, `IGNORE_ENTITY` — `javap` the enum and pick the match for your old value).
- Composting is now the **`Compostable` data component** (`ResolvableInt` layers via a
  `ContextIntProvider`, set with `Item.Properties.compostable(key)`), not `ComposterBlock.COMPOSTABLES`
  (Forge, removed) or `CompostableRegistry` (Fabric, removed). NeoForge uses a
  `data_maps/item/compostables.json` data map.

### Codecs / misc
- `BlockStateProvider.CODEC` is now `Codec<Holder<BlockStateProvider>>` — use **`DIRECT_CODEC`** for an
  inline provider in a feature codec.
- **`BlockPos.findClosestMatch(center, xz, y, predicate)` was removed** — reimplement (cuboid scan,
  closest by `distSqr`).

### Rendering (retained-mode churn continues)
- `PoseStack.mulPose(Quaternion)` → **`rotate(Quaternion)`** (`mulPose` now only takes Matrix/Transformation).
- `SubmitNodeCollector.submitModel(...)` arg list changed (a trailing nullable dropped from the common overload).
- `ItemInHandRenderer` → **`FirstPersonHandsAndItemsRenderer`**, now fully render-state-based
  (`PlayerRenderState` + `FirstPersonHandsAndItemsRenderState`; item rendering via `ItemStackRenderState`).
  A first-person item mixin is a real rework — the `submitArmWithItem` target still exists but its params
  are render states now. `PlayerRenderState` is `AvatarRenderState` in some contexts.

### Datapack JSON — surfaces at world load, in waves (audit the whole pack, don't drip-fix)
- **BlockState keys renamed `"Name"`/`"Properties"` → `"id"`/`"properties"`** everywhere (features'
  `cannot_place_on` + state providers, structure processor lists, template pools). A no-property state is
  `{"id":"minecraft:x"}` or a bare string. Symptom: hang at *preparing for world creation* with
  `Not a string: {"Name":...}; No key id in MapLike`.
- **State-provider types dropped the `_state_provider` suffix**: `weighted_state_provider` → `weighted`,
  `simple_state_provider` → `simple`. Symptom: `Unknown registry key ... block_state_provider_type`.
- **Advancements**: the `recipe_unlocked` trigger key `"recipe"` → **`"recipes"`** (value stays a string).
- **Loot tables**: condition and function discriminators unified to **`"type"`** — `{"condition":"minecraft:survives_explosion"}`
  → `{"type":"minecraft:survives_explosion"}`, `{"function":"minecraft:set_count",...}` → `{"type":"minecraft:set_count",...}`
  (the `"conditions"`/`"functions"` *list* keys stay).
- **Configured features**: `data/<ns>/worldgen/configured_feature/*.json` → `data/<ns>/worldgen/feature/*.json`,
  flatten `{"type","config":{...}}` → `{"type", <fields inline>}`; `placed_feature` id references are unchanged.
- **Recipes** were already 26.2-format (flat ingredients, `result:{id,count}`) — no change 26.2→26.3.
  Always diff against vanilla before assuming a type changed.

## Toolchain gotcha (brand-new MC versions)
Architectury Loom's bundled ASM can lag a just-released MC version: `transformProduction<Loader>` throws
`Unsupported class file major version N` when TinyRemapper reads a **Java-N class in the loader's
classpath** that the old ASM can't parse. For 26.3 the culprit was **LWJGL 3.4.3's `META-INF/versions/27/`
(Java-27) multi-release classes** on the NeoForge/Forge classpath — Fabric built because it resolved LWJGL
3.4.1 (tops out at `versions/25`). Find it by scanning jars for `META-INF/versions/<N>/` (not just the
first class). Fixes, cheapest first:
- **Pin the offending runtime-provided lib down** for the build via `resolutionStrategy.eachDependency`
  (LWJGL is never bundled in a mod jar, so a build-only downgrade is safe). This is what unblocked 26.3.
- Or bump architectury-loom / architectury-plugin to a build new enough to bundle a Java-N-capable ASM
  (often a snapshot right after a major MC release). Your mod CODE compiling is independent of this step.

**Rule:** a same-era bump is a `javap`-guided compile loop — copy, bump+verify versions, compile common,
look up each break in the target jar, then loaders, then audit the datapack whole-hog at world-load.
Defer cosmetic reworks; expect the toolchain to lag a brand-new version independently of your code.
