# Official docs, per-loader toolchains & the NeoForge primer chain

The authoritative sources for 2026+ modding, and the canonical toolchain/conventions each loader
documents. Use this to (a) cite the right official page, (b) set up the correct toolchain per loader,
and (c) get the **semantic** vanilla deltas `javap` can't reveal — from the NeoForge primer chain,
which even Fabric's own porting guide defers to. Verified against the live docs 2026-10-05.

## Canonical toolchain & conventions per loader

| Loader | Build plugin | Dev mappings | Metadata file | Entrypoint | Registration idiom |
|---|---|---|---|---|---|
| **Fabric** | **Fabric Loom** | mojmap (Yarn EOL — see below) | `src/main/resources/fabric.mod.json` | `main`→`ModInitializer`, `client`→`ClientModInitializer` | direct `Registry.register(BuiltInRegistries.X, key, obj)`; `props.setId(key)` at construction |
| **Forge** (LexForge) | **ForgeGradle 6.x** (from the MDK) | **`official` channel = Mojang names, NO param names** (dev); SRG at runtime | `META-INF/mods.toml` | `@Mod("id")` class, bus via `context.getModEventBus()` | `DeferredRegister` + `RegistryObject` on the mod bus |
| **NeoForge** | **ModDevGradle (MDG)**, plugin id `net.neoforged.moddev` (NeoGradle = legacy alt) | mojmap, optional Parchment params | **`META-INF/neoforge.mods.toml`** (renamed from Forge's `mods.toml`) | `@Mod` class, `IEventBus` ctor injection | `DeferredRegister` + `DeferredHolder` + `RegisterEvent`; mod-bus vs game-bus split |
| **Multiloader** | **Architectury Loom** over all three | mojmap (no-remap for 26.x) | per-loader (above) from `common` | per-loader thin entry → `common` | shim the registry API once (see playbook §3) |

- **NeoForge version == its MC version** (NeoForge 26.3.x targets MC 26.3). Pull the **current MDG
  version live** from `projects.neoforged.net/neoforged/ModDevGradle` — the docs example pins a stale
  `1.0.11`; don't hardcode it. Project bootstrap: the **NeoForge Mod Generator** ZIP.
- **Fabric** scaffolds from the **web template generator** (`fabricmc.net/develop/template`; Advanced
  Options: Kotlin / Kotlin buildscripts / datagen — no Yarn-vs-mojmap toggle, because mojmap is the
  only choice now). Creative tab: `CreativeModeTabEvents.modifyOutputEvent(...)` (verified). Items
  need **two** JSONs: `models/item/<n>.json` + `items/<n>.json` (missing the `items/` def = purple).
- **Forge**: `modLoader="javafml"`, `loaderVersion` tracks the Forge major; run configs via
  `genEclipseRuns`/`genIntellijRuns`/`genVSCodeRuns`.

## Mappings reality for 26.x

- **mojmap everywhere, Yarn is EOL.** Mojang shipped the first unobfuscated MC (1.21.11) and the
  Fabric Project **stopped maintaining Yarn from that point** — so every 26.x Fabric project is mojmap
  and carries **no `yarn_mappings` key**. (`docs.fabricmc.net/develop/porting/mappings/`)
- **Porting an existing Yarn mod to 26.1+** is a real documented step, not a hand-rename: Loom's
  **`migrateMappings`** task (no Kotlin support) or the **Ravel** IntelliJ plugin (Kotlin-capable).
  Both imperfect; worst on Mixins — review the output.
- **Forge dev vs runtime:** modern ForgeGradle's default **dev** channel is `official` (Mojang
  method/field names, **no** parameter names) — set `mappings channel: 'official', version:'<mc>'`.
  The SRG `func_#####`/`field_#####` names are the **runtime** mapping, applied to the release jar by
  `reobf`. So in a Forge dev workspace you see Mojang names, *not* SRG. Parchment adds param names on
  any loader.

## Java ladder (verify against the primer, not the stale install table)

- **1.20.2–1.20.4 → Java 17 · 1.20.5–1.21.11 → Java 21 · 26.x → Java 25.** The JDK bumped 21→25 at
  the `1.21.11 → 26.1` boundary (per the 26.1 primer). ⚠ NeoForge's *user* install table
  (`docs.neoforged.net/user/docs/`) still reads "1.20.5-latest = Java 21" and is **stale** — trust
  the primer's Java 25 for 26.x.

## The NeoForge primer chain — authoritative cross-loader deltas

Calendar versioning replaced the 1.x line: the chain is **1.21.11 → 26.1 → 26.2 → 26.3** (26.1 is
the first new-scheme release; you migrate *to* it *from* 1.21.11 — 26.1 ≠ 1.21.11). Read the primer
for your jump (`docs.neoforged.net/primer/docs/<ver>/`); the headline reworks `javap` won't explain:

**26.1 (`/primer/docs/26.1/`) — data-side reworks (break datagen/loot/recipe code):**
- **Loot "type unrolling":** all wrapper `*Type` classes removed (`LootPoolEntryType`,
  `LootItemFunctionType`, `LootItemConditionType`, `*NumberProviderType`, `FloatProviderType`,
  `IntProviderType`…); registries hold `MapCodec` directly; `getType()`→`codec()`, `CODEC`→`MAP_CODEC`.
  `FloatProvider`/`IntProvider` are now interfaces with record subtypes (`min()`/`max()`).
- **Villager trades** moved to datapack registries `VILLAGER_TRADE`/`TRADE_SET`
  (`data/<ns>/villager_trade/*.json`, `trade_set/*.json`); `addOffersFromItemListings()`→`addOffersFromTradeSet()`.
- **`RecipeSerializer` is now a record** `new RecipeSerializer<>(MapCodec, StreamCodec)` — inner
  Serializer classes gone; results are immutable `ItemStackTemplate`; `Recipe#assemble()` drops the
  `HolderLookup.Provider`. Renames: `TippedArrowRecipe`→`ImbueRecipe`, `ArmorDyeRecipe`→`DyeRecipe`.
- **Lazy data components:** attach to Holders at resource-load; `Holder#areComponentsBound()`;
  `Item.Properties.delayedComponent()`; `EitherHolder` removed; new `Validatable`/`ValidationContext`.
- **Per-dimension time:** `WorldClock`/`ClockManager`; `Level#getDayTime()`→`getOverworldClockTime()`.
  SavedData refactor: `DimensionDataStorage`→`SavedDataStorage`, `SavedDataType` keyed by `Identifier`.

**26.2 (`/primer/docs/26.2/`) — the render overhaul (matches our §5 cookbook):**
- `MultiBufferSource`/`Tesselator` **removed** → the SubmitNode feature-render pipeline (`SubmitNode`,
  `FeatureRenderPhase`, `FeatureRenderer`). Also gone: `OutlineBufferSource`, `BedRenderer`,
  `ShapeRenderer`→`ShapeOutlineFeatureRenderer`. `Font#drawInBatch*`→`prepareText()`/`PreparedText`.
- **`ChatFormatting` gutted** → format via `Style` (`withColor`/`withBold`/…); `TextColor` holds the
  color constants.
- **GPU/Vulkan groundwork:** `TextureFormat`→`GpuFormat` (`RGBA8`→`RGBA8_UNORM`); `VertexFormat`
  redesigned (≤16 elements, `Mode`→`PrimitiveTopology`, `Builder#add`→`addAttribute`);
  `LevelRenderer` split into `LevelRenderer`+`LevelExtractor`; advancement criteria/predicates
  repackaged to `net.minecraft.advancements.triggers`/`.predicates`.

**26.3 (`/primer/docs/26.3/`) — renderpearl + SDL + datapack registries:**
- **`com.mojang.blaze3d.*` render classes → `com.mojang.renderpearl.*`** (`GpuBuffer`/`GpuTexture`
  now interfaces); shader `#moj_import`→`#include`; OIT (order-independent transparency) added.
- **Input GLFW→SDL:** key/mouse codes changed (`KeyEvent.key` = SDL scancode; `keycode` = SDL
  keycode); text input via `TextInputManager`; `InputConstants$Type#SCANCODE` removed.
- **Reloadable datapack registries + data-provider rework:** separate `DataProvider` classes removed
  → `SingleRegistryBootstrap`/`MultiRegistryBootstrap`; `ContextAwarePredicate`→`Holder<LootItemCondition>`;
  `EntityRenderDispatcher` no longer takes `Minecraft`; `PoseStack#mulPose`→`rotate` (+`rotateDegrees`).

## Official reference links

- **Fabric:** docs `https://docs.fabricmc.net/` · develop `/develop/` · first item (idiom)
  `/develop/items/first-item` · mappings/Yarn-EOL `/develop/porting/mappings/` · port guide
  `/develop/porting/` · template `https://fabricmc.net/develop/template/`
- **Forge:** getting started `https://docs.minecraftforge.net/en/latest/gettingstarted/` · mod files
  `/gettingstarted/modfiles/` · FG6 mappings `https://docs.minecraftforge.net/en/fg-6.x/configuration/`
  · downloads `https://files.minecraftforge.net/net/minecraftforge/forge/`
  ⚠ **Official Forge docs cover only 1.20.x–1.21.x — there is no 26.x doc branch.** 26.x Forge facts
  (version map, loaderVersion) come from the maven/files site and are **unofficial**.
- **NeoForge:** hub `https://docs.neoforged.net/` · getting started `/docs/gettingstarted/` · mod files
  `/docs/gettingstarted/modfiles/` · registries `/docs/concepts/registries/` · MDG toolchain
  `/toolchain/docs/plugins/mdg/` · Parchment `/toolchain/docs/parchment` · **primers**
  `/primer/docs/` (and `/26.1/`, `/26.2/`, `/26.3/`) · user/install `/user/docs/` (Java table stale).

**Rule:** official docs are the authority for toolchain + conventions (Loom / ForgeGradle+MDK /
ModDevGradle); the **NeoForge primer chain is the authoritative source for per-version vanilla
deltas** — read it for your jump, then confirm signatures with `javap`. Don't trust the NeoForge
user-doc Java table (stale) or expect official Forge docs to cover 26.x.
