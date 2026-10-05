# Minecraft Mod Porting Playbook

A full-stack, reusable guide for porting **any** Minecraft mod to a new game version and/or
loader — Forge → Fabric, NeoForge → Fabric, an older Fabric build forward, and so on. The
methodology (decision tree → toolchain verification → cross-loader mapping → codemod discipline
→ layered validation → centralized crash guards → difficulty tiering) is version-agnostic.

The concrete technical detail throughout uses the **1.21.x / Forge / NeoForge → Minecraft 26.2
Fabric** jump as its worked example, because it is a large, representative transition (vanilla
API churn *and* a loader change *and* a rendering-pipeline rewrite all at once). When you port
across a different delta, the *specifics* change but the *structure* of the work does not — read
each section's closing **Rule** as the reusable takeaway.

> How to use this guide: start at §1 and triage every mod *before* writing code — most mods never
> need a source port. Only the handful that reach the bottom of the tree cost real time; §10 tells
> you which ones and why.

---

## 1. Port decision tree

Run this per mod **before writing a line of code.** Most mods exit at rung 1–4; only a few reach
rung 6–7.

```
For each source mod:
│
├─1. Does an OFFICIAL build for your TARGET version + loader already exist?
│     → USE IT. Done. (Typically the large majority of any pack.)
│
├─2. Does an official build exist on the OTHER loader?
│     ├─ Willing to run a compat layer (e.g. Sinytra Connector + Forgified Fabric API
│     │    to run NeoForge jars on Fabric)? → consider it (§3), but it is all-or-nothing
│     │    for the whole pack and adds a heavy, version-sensitive layer.
│     └─ Else → treat as "no native build" → rung 4/5.
│        ⚠ Some mods are single-loader-ONLY with no feasible path short of the compat layer.
│
├─3. Is this mod a BACKPORT of a feature that is now VANILLA in your target version?
│     → DELETE it. Do not port. (Verify the feature is actually present in the target jar first.)
│
├─4. Is there a NATIVE equivalent / refork for your target loader that does the same job?
│     → SWAP it. Cheaper than porting and already maintained.
│     (e.g. a Forge mod that has a "Refabricated" Fabric sibling; Curios → Accessories;
│      a feature vanilla absorbed → drop the standalone mod.)
│
├─5. Does the mod's own repo have a multiloader / newer-version branch?
│     → This is a LOADER-GLUE or small-bump port, not a full version lift. Far cheaper (§3).
│     ALWAYS check the repo before assuming a big lift — this is the most common way to turn
│     a "weeks" job into a "hours" job.
│
├─6. Is there published SOURCE (any version)?
│     → SOURCE-PORT it (§4–6). Cost scales with the version delta and the GUI/render surface.
│
└─7. No source, no native build, no equivalent, feature not vanilla?
      → DROP it, and tell the user exactly what is lost (a concrete "what you lose" list).
```

**Rule:** the decision tree converts "N mods to port" into "a fraction to actually touch, of which
a handful are hard." Spend your budget on rung 6–7; never hand-port a rung-1/3/4 mod.

---

## 2. Toolchain / environment

### Step zero: determine how your target version is mapped at runtime.
The single fact that changed everything for the 26.2 era: **26.2 Fabric ships mojmap (official
Mojang names) at runtime, not intermediary.** Older Fabric versions shipped intermediary
(`class_####` / `method_####`) and required a `remapJar` step; 26.2 does not.

- **Verify any jar's mapping:** extract it and scan the class bytes for intermediary tokens:
  `grep -aoE "class_[0-9]{3,}|method_[0-9]{4,}" <extracted-classes> | wc -l` — `0` = mojmap
  (correct for 26.2), nonzero = intermediary (needs remap). ⚠ Scan *extracted* class files; raw-jar
  grep misses DEFLATE-compressed bytes. This token grep detects **Fabric intermediary only**.
  Forge *dev* uses ForgeGradle's `official` channel (Mojang names, no param names, since 1.14+) with
  SRG `func_`/`field_` applied only at *runtime* by `reobf`; NeoForge *dev* is mojmap via ModDevGradle
  (NeoGradle = legacy alternative). So for a Forge/NeoForge target you see Mojang names in dev — check
  that toolchain's mapping, not this Fabric grep; the settle-first rule is the same.
- **Build implication (26.2):** compile against official Mojang mappings with **no refmap** and
  **no intermediary `remapJar` step** — the plain named jar is already runtime-correct. For an
  older-Fabric target, the opposite holds (keep intermediary + remap).

Whatever your target, nail this down first — it dictates your entire Gradle/Loom config.

### Loom setup that builds (26.2 example, adjust versions to your target)
- A current Gradle + fabric-loom pairing that resolves your target (for 26.2: Gradle 9.x,
  fabric-loom 1.16+/1.17-SNAPSHOT).
- fabric-loader new enough for your other plugins (architectury/kotlin needed ≥0.19.5 in the 26.2
  era), matching fabric-api, cloth/modmenu if used.
- The JDK the target requires (26.2 → **JDK 25**). Note `javap`/`javac` may not be on PATH in
  git-bash — call them by full JDK path.
- Official mappings, **no `mappings()` line on a single module**, mixin json with **no `refmap`
  key**, `compatibilityLevel` set to the source's Java level.
- Prefer the **lowest bytecode `--release` that runs on the target runtime** (e.g. `--release 21`
  bytecode runs fine on a Java 25 runtime) — only raise it when the source uses newer language
  features.

### Multiloader (Architectury) gotcha
The thin `:fabric:jar` holds only loader-specific classes. The complete mod (common + assets
merged) is the **shadowJar** output. Deploy that, renamed — not the thin jar.

### Mod-metadata essentials (recurring boot failures)
- Depend on the correct API key for your target (in fabric-api, the legacy `fabric` alias was
  replaced by `fabric-api`; a stale alias = `HARD_DEP_NO_CANDIDATE` at boot).
- Widen the `minecraft` version predicate (e.g. `">=26.1 <27"`) — an exact or template range can
  *exclude* the real runtime version.
- Jar-in-Jar any bundled config/util lib whose setup runs at init, or it crashes.

### Rebuilding a jar-only mod (no source)
Decompile with **Vineflower** (clean Fabric-friendly output), scaffold a Loom project from a
known-good target build, drop the decompiled sources + resources back in, add compile-only deps.
Decompiled `(FunctionalInterface)arg -> …` casts are valid — leave them.

**Rule:** the mapping/runtime model is the whole build model. Settle it first; if a dependency jar
leaks the wrong mapping, re-remap *that jar* (§4) rather than fighting it at the consumer.

---

## 3. Cross-loader strategy (Forge / NeoForge → Fabric)

### Three fundamentally different jobs — identify which you have
1. **Old Forge (e.g. 1.20.1) → new Fabric (26.2)** = a *version lift* (the biggest: vanilla APIs
   changed under you **and** the loader changed).
2. **Current NeoForge → current Fabric (same MC version)** = *loader-glue only* (vanilla APIs
   already correct; only `@Mod`/event-bus/registries/networking/config change). **Much cheaper** —
   always prefer finding this branch.
3. **Old Fabric → new Fabric** = a *same-loader version bump* (only the vanilla-API delta, no
   loader work).

### Loader-glue conversion map (NeoForge/Forge → fabric-api, verify each against your target jar)
| NeoForge/Forge | Fabric |
|---|---|
| `@Mod` + ctor | `implements ModInitializer` / `ClientModInitializer` (entrypoints in mod metadata) |
| `DeferredRegister` / `registerItem` | `Registry.register(BuiltInRegistries.ITEM, id, new Item(props.setId(key)))` |
| `AttachmentType` / capability | `fabric-data-attachment-api-v1`; access via `((AttachmentTarget)e).getAttachedOrCreate(TYPE)` (cast) |
| `BuildCreativeModeTabContentsEvent` | `fabric-creative-tab-api-v1` `CreativeModeTabEvents.modifyOutputEvent(key)` |
| `PayloadRegistrar.playToClient` | `PayloadTypeRegistry.clientboundPlay().register`; send `ServerPlayNetworking.send(player,payload)` |
| `RegisterCommandsEvent` | `CommandRegistrationCallback.EVENT` |
| `PlayerEvent.Clone` | `ServerPlayerEvents.COPY_FROM(old,new,alive)` (`!alive` == `isWasDeath()`) |
| `ItemTooltipEvent` | `ItemTooltipCallback.EVENT` |
| `ModConfigSpec` | self-contained Gson POJO in `FabricLoader.getConfigDir()` (Gson is on MC's classpath) |
| `FMLEnvironment...isClient()` | `level.isClientSide()` / `player.isLocalPlayer()` guard |
| `ModList.get().isLoaded(id)` | `FabricLoader.getInstance().isModLoaded(id)` |
| `neoforge:difference` recipe ingredient | `fabric:difference` (identical strings; discriminator key is `fabric:type`) |

### The big reusable shim: a registry-API shim
Large Registrate/NeoForge content mods are written against a `DeferredRegister`/`RegistryObject`
DSL over hundreds of call sites. **Don't rewrite the call sites — shim the registry API once:**
1. A `RegistryObject<T>`/`DeferredHolder<T>` shim class (holds the id + a resolved `T`;
   `.get()`/`.getKey()`/`.getId()`). **It CANNOT `implements Holder`** — vanilla seals `Holder`
   in 26.2 (Forge/NeoForge un-seal it; Fabric does not). Expose `value()`/`get()` instead.
2. A helper `createItem/createBlock/...` that does eager `Registry.register(...)` and returns the shim.
3. A `Block.Properties`/`Item.Properties`-ctor **mixin** that reads a `ThreadLocal<ArrayDeque<ResourceKey>>`
   (the id currently being registered) and calls `.setId(...)` automatically — so bare
   `new Item.Properties()` call sites compile unchanged. **The ThreadLocal MUST be a per-thread
   stack (ArrayDeque), not a single slot** — nested registration (a block's Properties touching a
   sound-registry constant mid-construction) wipes a single slot and NPEs the outer ctor.
4. Datagen classes never run at runtime — drop their framework deps; the generated JSON already ships.

### A Forge-parity shim lib (e.g. porting_lib)
Shim libraries re-expose Forge's DeferredRegister, loot-modifier, block/level-event, item-ability,
and fluid-container APIs on Fabric, which lets heavy Forge-API mods compile. **Caveat:** such libs
are built on string-targeted mixins, so a module compiles green but its mixins fail in waves at
datapack/world-load as each stale `@Shadow`/`@At` surfaces. When a mixin targets a removed/renamed
member **and** the feature is unused by the consuming mod, **drop it from its `*.mixins.json`**
(honest, matches the source-port drop pattern).

### Compat layer vs a true port
A compat layer (Sinytra Connector + Forgified Fabric API) lets official NeoForge jars run on
Fabric, but **it is all-or-nothing for the pack** — a heavy, version-sensitive layer. A true port
keeps the stack pure. Keep the compat layer in your pocket for the 1–2 mods that are genuinely
single-loader-bound; port everything else.

**Rule:** before any cross-loader port — (a) check the repo for a multiloader branch to downgrade a
lift into glue, (b) check for a native refork to skip it entirely, (c) reach for the registry shim
before touching call sites, and (d) accept a compat layer only for the stragglers.

---

## 4. Recurring API changes + codemods (26.2 cheatsheet)

The high-frequency mechanical changes for the 1.21→26.2 delta, as a table you can codemod. For a
different delta, build the equivalent table by diffing the two vanilla jars — the *approach* (split
renames from reworks, codemod the renames, hand-judge the reworks) is what transfers.

> **Authoritative cross-loader delta source:** the **NeoForge primer chain**
> (`docs.neoforged.net/primer/docs/`, chain 1.21.11→26.1→26.2→26.3) documents each version's vanilla
> breaking changes, and Fabric's own port guide defers to it rather than listing deltas. Read the
> primer for your jump to catch the *semantic* reworks `javap` can't show (26.1 loot type-unrolling /
> datapack villager trades / RecipeSerializer-as-record / per-dimension `WorldClock` / lazy data
> components; the 26.2 render overhaul; 26.3 Blaze3d→renderpearl + GLFW→SDL). See
> [official-docs-and-primers.md](official-docs-and-primers.md). The table below stays the hand-derived
> quick-codemod list for the 1.21→26.2 jump.

### Renames / moves (pure mechanical)
| Old (≤1.21) | New (26.2) | Note |
|---|---|---|
| `ResourceLocation` | `net.minecraft.resources.Identifier` | `Identifier.fromNamespaceAndPath(ns,path)` / `parse` — no `.of()` |
| `ResourceKey.location()` | `.identifier()` | but `TagKey.location()` STAYS |
| `EntityType.ARMOR_STAND` | `EntityTypes.ARMOR_STAND` | ALL_CAPS consts moved to `*s` holders; same for `BlockEntityTypes` |
| `Blocks.WHITE_TERRACOTTA` | `Blocks.DYED_TERRACOTTA.pick(DyeColor.WHITE)` | ColorCollection: DYED_TERRACOTTA/GLAZED/BED/DYED_CANDLE/CONCRETE/WOOL/… |
| `Blocks.WAXED_COPPER_BLOCK` | `Blocks.COPPER_BLOCK.waxed().unaffected()` | WeatheringCopperCollection |
| `Registry.get(id)` | `getValue(id)` | `get(id)`→`Optional<Holder.Reference>`; use `getValue` for the value |
| `RegistryAccess.registryOrThrow` | `lookupOrThrow` | |
| `Level.getMinBuildHeight` | `getMinY` | `BlockPos.getCenter()`→`Vec3.atCenterOf(pos)` |
| `Direction.getNormal()` | `getUnitVec3i()` | |
| `Entity.moveTo` | `snapTo` | ⚠ NOT `PathNavigation.moveTo` |
| `MobSpawnType` | `EntitySpawnReason` | |
| `UseAnim` | `ItemUseAnimation` | |
| `FastColor`/`ARGB32` | `net.minecraft.util.ARGB` | |
| `ChatFormatting` color meta | `net.minecraft.network.chat.TextColor` | `getById(i)`→`values()[i]`, `getId()`→`ordinal()` |
| `ItemInteractionResult` | merged into `InteractionResult` | `sidedSuccess`→`SUCCESS` |
| `DirectionProperty` | `EnumProperty<Direction>` | |
| `noCollission()` | `noCollision()` | |
| advancements `criterion.*` | `predicates.*` + `triggers.*` | package split |
| `MultiBufferSource` | submit pipeline (`SubmitNodeCollector`) | render, §5 |

### Removed / replaced (needs rework, not rename)
- **`GuiGraphics` → `GuiGraphicsExtractor`** (retained-mode) — §5, the big one.
- **`Explosion` is now an INTERFACE** — custom explosions `extends Explosion` → `implements
  Explosion` or extend `ServerExplosion` (no `this.x/level/radius`, no `getToBlow()`/`finalizeExplosion`).
- **`Block` LOST `appendHoverText`** entirely (only `Item` has it). `Item.appendHoverText` sig =
  `(ItemStack, TooltipContext, TooltipDisplay, Consumer<Component>, TooltipFlag)`.
- **`Entity.hurt` is final** → override `hurtServer(ServerLevel, DamageSource, float)`. Same pattern:
  `customServerAiStep(ServerLevel)`, `doHurtTarget(ServerLevel,…)`.
- **NBT: `CompoundTag` in save/load → `ValueInput`/`ValueOutput`.** BE
  `loadAdditional/saveAdditional(ValueInput/ValueOutput)`; Entity `readAdditionalSaveData(ValueInput)`.
  Getters return Optional or `getIntOr(k,def)`. `putUUID`/`getUUID` removed → `store/read(key,
  UUIDUtil.CODEC)`; `NbtUtils.writeBlockPos` → `store(key, BlockPos.CODEC)`. ⚠ On-disk layout differs
  → target a fresh world, not backward-compatible.
- **`SpawnEggItem` ctor** dropped colors → `(Properties)` only; bind entity via
  `Item.Properties.spawnEgg(EntityType)`; colors are a client tint.
- **`RecordItem` removed** → `JukeboxSong` datapack + `JUKEBOX_PLAYABLE` component. `ArmorItem`
  removed → `EQUIPPABLE` component. `ItemNameBlockItem`/`BannerPatternItem`/`EitherHolder`/`VariantHolder`
  removed.
- **`Ingredient` is final** (can't subclass); `.getItems()`→`items()`; `Ingredient.of(ItemStack)`
  gone → `of(ItemLike)`; tag ingredient binds eagerly → memoize in a `Supplier`.
- **`new ItemStack(Item)` / `Item.getDefaultInstance()` reads unbound components during recipe
  decode / `<clinit>`** → "Components not bound yet". Defer ALL recipe-result/static `ItemStack`
  construction to runtime (lazy `Supplier<ItemStack>`). **Never build an ItemStack in a `<clinit>`
  of any class that can load during resource reload** (BERs, entity renderers, models, particles) — §8.
- **26.2 food rework:** `.food(...)` no longer drives eating → add `.component(DataComponents.CONSUMABLE,
  Consumable.builder()...build())`.
- **Item id MUST be on Properties at construction:** `new Item(new Item.Properties().setId(
  ResourceKey.create(Registries.ITEM, id)))` or the ctor throws `NullPointerException: Item id not
  set` (block: `Registries.BLOCK`). Prime codemod target.

### Recipe/datapack JSON (silent — surfaces in client/data log, not boot)
- **Ingredient format:** `{"item":"minecraft:stick"}` → `"minecraft:stick"`; `{"tag":"x"}` →
  `"#x"`; multi-option → JSON array. Applies to shaped `key` + shapeless `ingredients`. Result
  stays `{"id":"...","count":N}`. Symptom: recipe silently doesn't exist.
- **`minecraft:random_patch` FEATURE removed** → unwrap scattering into placed_feature placement
  modifiers. **`uniform` IntProvider flattened** (drop the `value` wrapper). **`minecraft:grass`
  block → `minecraft:short_grass`** *(only for pre-1.20.3 sources; a 1.21.x source is already
  `short_grass`)*. **`TreeConfiguration` dropped `dirt_provider` → requires
  `below_trunk_provider`.**

### Codemod discipline
- **Always `clean compileJava`, never trust an incremental error count.** javac short-circuits per
  file: unresolved IMPORTS hide BODY errors, so the count ticks UP as imports resolve, and a
  mangled statement (a dangling `)`) aborts early printing a FAKE low count. Confirm progress by
  emitted `.class` count or a clean compile — never by a shrinking error log.
- Keep codemods as small, re-runnable scripts (one per transformation: id-setter, recipe-format,
  rename-table), checked against a clean compile after each pass.

**Rule:** rename-table changes are codemod fodder; removed/replaced ones need per-file judgment.
Measure with clean compiles only.

---

## 5. GUI / render cookbook (GuiGraphics → GuiGraphicsExtractor)

26.2 (1.21.6+) replaced immediate-mode `GuiGraphics` with a **retained-mode extract/render**
pipeline. `GuiGraphics` is GONE. Every GUI-heavy mod hits this identically.

**Paradigm:** widgets/screens no longer DRAW; they EXTRACT render state into a
`GuiGraphicsExtractor`, and `GuiRenderer` renders it. `render(GuiGraphics)` bodies port mostly 1:1;
the **signature** and a few calls change.

**Signature map (before → after):**
- `Renderable.render(GuiGraphics,int,int,float)` → `extractRenderState(GuiGraphicsExtractor,int,int,float)`.
- `AbstractContainerScreen` overrides: `renderBg`→`extractBackground`, `renderLabels`→`extractLabels`,
  `renderSlot`→`extractSlot`, `renderTooltip`→`extractTooltip`.
- `g.drawString(Font,t,x,y,c)` → `g.text(Font,t,x,y,c[,shadow])`; `drawCenteredString` → `centeredText`.
- `g.blit(Identifier,x,y,u,v,w,h,…)` → `g.blit(RenderPipeline, Identifier, x, y, float u, float v,
  w, h, texW, texH[,color])` — pass `RenderPipelines.GUI_TEXTURED` as arg 1. `blitSprite` likewise.
- `g.renderItem(stack,x,y)` → `g.item(stack,x,y[,seed])`; `renderFakeItem`→`fakeItem`.
- **`g.pose()` returns `org.joml.Matrix3x2fStack` (2D)** — `pushMatrix()/popMatrix()/translate/scale/rotate`,
  no z. `pushPose/popPose/last().pose()` don't exist.
- **Tooltips deferred:** `renderTooltip(...)` → `setTooltipForNextFrame(font, stack|Component|List, x, y)`.
- **Mouse/key events:** `mouseClicked(double,double,int)` → `mouseClicked(MouseButtonEvent, boolean
  doubleClick)`; `keyPressed(int,int,int)` → `keyPressed(KeyEvent)`. Extract `event.x()/y()/button()`
  at the top to reuse bodies.
- **Container clicks:** `ClickType` removed → `ContainerInput` (maps 1:1).
- `imageWidth`/`imageHeight` are now **final** → add an access-widener `mutable field` line.

**fabric-api removals (need rework, not rename):**
`BlockRenderLayerMap` (render_type is data-side), `BuiltinItemRendererRegistry`
(→`SpecialModelRenderer`), `ColorProviderRegistry`/`ItemColor` (→ data-driven `ItemTintSource`),
`LivingEntityFeatureRendererRegistrationCallback` (→`FeatureRendererRegistry`), `HudRenderCallback`
(→`HudElementRegistry.addLast`), `ClientPickBlockApplyCallback` (no successor).
`EntityModelLayerRegistry`→`ModelLayerRegistry`. `Minecraft.screen` field REMOVED → `mc.gui.screen()`.
`ExtendedScreenHandlerType` → `net.fabricmc.fabric.api.menu.v1.ExtendedMenuType`.

**In-world custom geometry — the submit pipeline** (separate from GUI; harder):
`MultiBufferSource` is gone. BER/EntityRenderer is now `createRenderState()` →
`extractRenderState(...)` → `submit(state, PoseStack, SubmitNodeCollector, camera)`. Template on
vanilla **`BeaconRenderer`** for raw geometry. Bridge: `SubmitNodeCollector.submitCustomGeometry(
matrix, RenderType, (pose,buffer) -> …)` returns a real `VertexConsumer` and COPIES the pose — seed
a fresh local `PoseStack` from it and your existing `ModelPart.render(...)` runs verbatim. **Wrap
every `submit`/`extractRenderState` body in try/catch** — the dispatcher rethrows as a HARD crash;
degrade to "not drawn + one log line." Reusable helper layers (e.g. Moonlight's `RenderUtil`
`submitBlockModel`/`getBlockModel`) can breach most of this for block-model geometry.

**Picture-in-picture entity previews (an entity rendered in a GUI slot).** The retained-mode GUI
renders entity previews deferred, through a single shared `GuiEntityRenderer` that renders each into
one reused GPU texture and records a *deferred* blit. If several previews are submitted in one frame,
on **Forge** all blits read the final texture → every preview collapses into the last entity
(Fabric/NeoForge tolerate it). Fix: give each preview its **own** picture-in-picture renderer (own
texture) via the loader's register hook, so no renderer renders more than one preview per frame.

**Worked progress metric:** a GuiGraphicsExtractor codemod across a large storage mod's ~40 GUI
files took it from ~620 errors to ~170 in one pass — voluminous but tractable, NOT a from-scratch reimpl.

**Rule:** the GUI port is a mechanical signature pass (extract-* + call renames + 2D pose). The
in-world submit pipeline is a genuine reimpl — template on BeaconRenderer and always try/catch.

---

## 6. Crash-guard pattern library

Centralize every runtime crash guard into **one or two standalone mixin mods** rather than editing
each content jar — the guard mod survives every content rebuild, builds with no dependency on the
target mod, and deploys as a single jar. *(This assumes a mixin-capable loader — Fabric, or
Forge/NeoForge via a mixin coremod. The `"*"`/`"client"` env below is Fabric's key; Forge/NeoForge
use dist checks + the mixin-json `client` array. Pre-1.12.2 Forge predates standard Mixin and uses
ASM coremods / Forge event hooks instead.)*

- **Make its environment `"*"`, not `"client"`.** A client-only env means Fabric skips it on a
  dedicated server, so common crash guards never apply server-side (a frequent cause of "the
  ticking crash I fixed keeps coming back").
- Each guard is a **symptom → root cause → mixin** recipe. The table below is a catalogue of the
  shapes that recur across 26.2 ports:

| # | Symptom | Root cause | Guard |
|---|---|---|---|
| 1 | `IllegalStateException: has not defined synched data value 8` on block PLACEMENT (server crash) | A ported entity's `defineSynchedData` override skips `super`, leaving an inherited accessor id undefined (common on `HangingEntity` subclasses) | A mixin that defines the missing synched-data id for the subclass |
| 2 | `NoSuchElementException` / "No value present" at entity construction | `registry.get(KEY).orElseThrow()` on an empty/unshipped datapack registry | `@Redirect` the lookup → `getAny()` fallback |
| 3 | Client StackOverflow (via chunk build) | An emissive-rendering override whose body calls itself (`return state.emissiveRendering()`) → infinite recursion | `@Inject HEAD cancellable`, `setReturnValue(false)` |
| 4 | Server-tick crash — `Cannot set property … on <block>` | A worldgen/fluid generator calls `state.setValue(PROP,…)` on a block lacking that property | `@WrapMethod` the generator, catch `IllegalArgumentException` → `Optional.empty()` |
| 5 | "Ticking entity" — `Can't find attribute minecraft:tempt_range` | 26.2 `TemptGoal` reads a NEW attribute; ported mobs built their `AttributeSupplier` by hand without it | Guard `getValue`/`getBaseValue` HEAD, return `getDefaultValue()` when the attribute is absent |
| 6 | `AttributeMap.getValue … supplier is null` on spawn | Entity registered with NO default AttributeSupplier | Same guard for `supplier == null` |
| 7 | `IllegalStateException: Invalid block entity … got Block{X}` on placement/chunk-load | A mod adds a block variant but not to the BE type's `validBlocks` | `@Inject` RETURN of `isValid`, accept when the block's class ==/isAssignableFrom any `validBlocks` member |
| 8 | "Unknown TooltipComponent" crash hovering an item | A data-side `TooltipComponent` with no client renderer | A `ClientTooltipComponentCallback` handler returning a zero-size empty tooltip for those classes. **Do NOT mixin `ClientTooltipComponent` — it's an interface** |
| 9 | Many entities crash `NPE EntityRenderer.shouldRender … renderer is null` | The port cut the client package but still registers the entity; no renderer | A client mixin that fills the renderer map with a `NoopRenderer` on reload |
| 10 | Mod "Failed to get registry access. This is a bug" → recipes silently uncraftable | The mod generates recipes off-thread at startup before any RegistryAccess is published | `@WrapMethod` the accessor to return a captured RA, else a lazily-built fallback |
| 11 | Recipe-book "can't be placed due to empty ingredients" | Hand-ported recipe classes stub `placementInfo() -> NOT_PLACEABLE` → 26.2 drops them | Inject `placementInfo()` returning `PlacementInfo.create(ingredient)`, or force `isSpecial() -> true` |

**When a NEW crash of these shapes appears:** grep the ported jars for the offending pattern (a
`defineSynchedData` override lacking `invokespecial super`, an `orElseThrow` on a registry lookup, a
`setValue` on a maybe-absent property), add the same mixin shape to the guard mod, rebuild, deploy
(game CLOSED — §8). Keep the target version's vanilla jar handy for signature checks.

**Offline build gotcha:** fabric-loader bundles Mixin/MixinExtras only as NESTED jars (javac can't
read those) → every `@Inject/@At` fails "cannot find symbol". Add the standalone Mixin + MixinExtras
cache jars to the classpath when building a guard mod outside Loom.

**Rule:** one centralized mixin mod beats editing N content jars. Make its env `"*"` so guards cover
the dedicated server too.

---

## 7. Validation stack (run in this order)

Each stage catches a class of failure the previous one is BLIND to. Do not skip forward.

1. **Compile** — `clean compileJava` (clean, every time; §4 masking rule). A green compile proves
   nothing about mixins or runtime.
2. **Boot-verify on a dedicated server** — stand up an isolated server (API + core libs + the mod
   only), boot to `Done`. This is the authoritative COMMON-mixin validator — it fails fast on the
   first bad common mixin with the exact target. Mixin faults surface in WAVES (a registration
   crash hides datapack-load mixins, which hide world-load mixins) → check the log after EVERY fix.
   Grep for `FAILED during APPLY | was not applied | Found a remappable @Shadow` — never trust
   boot-to-Done alone.
3. **Static mixin audit** — parse the merged jar's constant pool for `@Mixin/@Shadow/@Inject/
   @Accessor/@Invoker` and check each target exists. **NAME-LEVEL ONLY** — it catches
   target-class/field/method-name breaks but MISSES `@At` call-site + descriptor mismatches. A first
   pass, not a verdict.
4. **⚠ CLIENT mixins are a BLIND SPOT for server boot** — a dedicated server never loads the
   `"client"` array. A server-green port can still crash-cascade on the real client, one stale
   client-mixin at a time (each APPLY failure is fatal). Validate EVERY client mixin statically
   against the merged jar, or test on a real client.
5. **Bake gate (purple/invisible)** — a script that resolves blockstate → model → parent → textures
   the way the baker does, across the layered resource view. It flags: missing blockstate/itemdef, a
   model with a Forge `"loader":` field and no vanilla geometry, a parent chain with no geometry,
   `parent:"builtin/entity"` (removed in 26.2), or a missing non-particle texture. This is the
   objective QC gate for any icon/model fix. It does NOT catch runtime COLOR/tint failures (bake OK,
   wrong color).
6. **Creative-inventory icon test** — 26.2 REQUIRES a separate `assets/<ns>/items/<name>.json` model
   *definition* (`{"model":{"type":"minecraft:model","model":"<ns>:item/<name>"}}`) in addition to
   `models/item/*.json`. Mods built pre-1.21.4 ship zero `items/` defs → all icons purple. Generate
   one `items/` def per `models/item/`.
7. **RCON phantom detection** — `execute if block <pos> <id>` parses the block arg *before* the
   position-loaded check, so an UNregistered id throws `Unknown block type` read-only in ~2s. The
   only reliable phantom detector (no offline heuristic works). For mobs, the prune signal is
   **"Can't find element"** on summon, not "Unknown entity".
8. **In-client QA** — the last mile; boot-to-Done NEVER exercises client rendering. Walk a full
   block grid and spawn every mob. A datapack harness that places every registered block (with a
   1-block gap, per-mod grids, doors/beds as whole multiblocks, forceload-tiled, default-gamerule-safe)
   plus a summon-all-mobs function makes this systematic. The live client `logs/latest.log` is the
   game's own bake-error report.

**Rule:** compile → server-boot → static audit → client-mixin check → bake gate → icon test → RCON
phantom scan → in-client. Three independent "it works" illusions live here (green compile,
boot-to-Done, asset-chain verify) — each is necessary and none is sufficient.

---

## 8. Common failure modes → fixes

| Symptom | Root cause | Fix |
|---|---|---|
| **Black screen** (render thread alive, CPU accruing, title never draws) | Startup resource-reload ABORTED → rollback. Grep `latest.log` for **`Caught error loading resourcepacks, removing all selected resourcepacks`**; the `CompletionException` after it names the cause | Fix the underlying reload exception (below). Bisect by emptying `mods/` → vanilla renders → add back upward (libs first). Check the log after EVERY fix — a second cause hides behind the first |
| Black screen via atlas stitch — `Dest texture … not large enough` | An animated-texture `.png.mcmeta` with an EMPTY `"frames": []` (legal pre-26.2, fatal in 26.2) | Delete the empty `frames` key (26.2 auto-detects from height) |
| Black screen / purple — `pack_format > 64` rejected | `pack.mcmeta` missing `min_format`/`max_format` | Add `min_format`/`max_format` ([major,minor] arrays). Legacy `supported_formats` does NOT satisfy it |
| Black screen — "Failed to load required shader programs" | A pre-26.2 core shader was renamed (e.g. `core/rendertype_text` → `core/text`) | Alias the shader (ship the old name as a copy of the new one) |
| `NullPointerException: Components not bound yet` on reload | An `ItemStack` built in a `<clinit>` of a class that loads during resource reload (BER/renderer/model) | Make it a lazy static getter. **Never construct ItemStack / `getDefaultInstance()` in a `<clinit>` of a reload-loadable class** |
| Item/block renders **purple or invisible when PLACED** (icon looked fine) | A Forge/NeoForge block model with a `"loader":` field → Fabric silently fails to bake → invisible/purple. Icons used opaque fallback textures so they masked it | Register a Fabric model loader OR flatten geometry to a plain vanilla `elements` model. The bake gate (§7) is the objective check |
| Backpack/tinted item renders GRAY | Relied on Forge/NeoForge tint sources that don't apply on Fabric → grayscale × white | Bake the tint into the texture (grayscale × color per-channel, alpha preserved), strip `tintindex`/`tints`; OR register a Fabric `ItemTintSource` |
| False "crash" — `ZipException: bad LOC / zip END header not found` | A jar was rewritten while the game was OPEN — live jars are locked/mapped | **Close the game before touching `mods/*.jar`.** Edit options.txt / mcmeta / jars only game-closed (MC rewrites options.txt on exit, clobbering live edits) |
| `ClassNotFoundException` at mixin PREPARE on client | mixins.json declares `client` mixin classes the port stripped | Remove the dangling `client` entries |
| `ClassNotFoundException` for an integration entrypoint when JEI/Jade present | mod metadata lists entrypoints for deferred compat classes not in the jar | Trim those entrypoints |
| `validateAccessWidener` fails the build | AW lines target members removed in 26.2 | Clean compile ⇒ no live code uses them ⇒ strip exactly those flagged line numbers |

**Rule:** a black screen is almost always a resource-reload abort, not a render-backend failure —
grep the one magic log line first. Everything "purple/invisible" is a bake failure (`"loader":`
field, missing `items/` def, `builtin/entity`), gated by the bake script. And never edit a jar with
the game open.

---

## 9. Distribution

**Distribute a modpack via a CurseForge EXPORT ZIP (with overrides), NOT the share/import CODE.**
- The code is manifest-only — it references CF file-ids = STOCK files, so it **silently drops every
  hand-ported mod** (anything with no real target-version CF release) AND every jar you baked a fix
  into.
- The export ZIP includes the **overrides** folder → it carries all jars 1:1 and is complete.
- Same trap applies to a server host: **upload the actual jar FILES**, not a CF export. CF-launcher
  "modified files" markers are your intentional edits and do NOT travel to the server.

**CurseForge "edited files" management:**
- The "this project's files were modified" flags are YOUR edits. **Never click Update / Reinstall /
  Update-All** — it reverts the fix.
- Audit them into **asset-only edits** (lang/texture/model/blockstate → movable into a resource pack,
  freeing the mod back to stock) vs **real code edits** (keep pinned).
- The flag is cached as `isModified` in `minecraftinstance.json` and NOT auto-cleared even after a
  byte-identical restore. To clear it without reinstalling: **edit `minecraftinstance.json` while
  CurseForge is fully CLOSED** — recompute CF's fingerprint (murmur2 seed=1 over the file bytes with
  `0x09/0a/0d/20` whitespace stripped) per addon; if it MATCHES the stored `packageFingerprint`, set
  `isModified:false` + each `modules[].invalidFingerprint:false` (genuinely-edited mods mismatch and
  correctly stay pinned).

**A "core fixes" resource pack is often a hard dependency for distribution** — it holds the baked
item-model defs, lang aliases, and model fixes that can't live in a stock jar. Push it to players
via `server.properties` `resource-pack=<URL>` + `resource-pack-sha1` (recompute the sha1 at deploy).

**Server host config when the loader/version changed:**
- Set the loader + MC version, the matching Java (26.2 → **Java 25**), and a modern GC (**g1gc**; a
  stale `concMarkSweep` flag stops the JVM from starting on Java 14+).
- **A fresh world is required** when worldgen/biome-source registries changed — an old world fails
  `IllegalStateException: Overworld settings missing` (the saved modded biome source won't decode).
  Rename `world/` to force regen.
- Deduplicate mods (scan each jar's mod id; clean = N jars / N distinct ids / 0 dups) — a stale CF
  manifest can reintroduce an older duplicate of a mod you pinned.
- Add the target-version performance stack (sodium/lithium/ferritecore/c2me/krypton/etc.); note some
  perf mods may have no build for a brand-new version yet — skip those.

**Rule:** export-zip-with-overrides or raw files only; the import code drops every hand-port. The
resource pack is a hard dependency for players. Never pre-deploy-update pinned versions.

---

## 10. Difficulty tiering (what makes a mod hard)

Tier every mod at decision time — the cost of a pack is dominated by the handful of Tier-4 mods.

| Tier | Effort | What it is |
|---|---|---|
| **0 — Trivial** | minutes | Official target build exists → just install |
| **0b — Delete** | minutes | Feature is now vanilla (backport) → drop |
| **0c — Swap** | minutes | Native equivalent / refork exists for the target loader |
| **1 — Loader-glue** | hours | Repo has a same-version branch on the other loader → only `@Mod`/events/registries/config change; vanilla APIs already correct |
| **2 — Asset-only edit** | hours | Code fine; just missing target-version `items/` defs, lang aliases, mcmeta, `grass`→`short_grass`, worldgen JSON migrations |
| **3 — Source port (self-contained)** | days | A version lift: API renames + NBT + maybe one GUI/render surface, but ONE module, no deep lib chain |
| **4 — Deep multi-mod port** | weeks / multi-session | A lib + its dependents, a sealed-Holder/registration-model rewrite, full GUI + in-world submit-pipeline render, cross-module mixin waves |

**What pushes a mod up the tiers (score these before committing):**
- **Registration model** — a Forge `DeferredRegister`/Registrate DSL over hundreds of call sites +
  26.2's `Properties.setId` requirement + sealed `Holder` = a framework-shim port *before* you touch
  content (Tier 4).
- **GUI surface** — any screen/widget drags in the entire GuiGraphicsExtractor retained-mode pass (§5).
- **In-world custom geometry** — BER/entity renderers against the removed `MultiBufferSource` = the
  submit-pipeline reimpl (the genuine wall; everything else is mechanical).
- **Mixin depth** — string-targeted mixins compile green then fail in waves at datapack/world/client-load;
  client mixins are a server-boot blind spot.
- **Dependency chain** — a shim lib with internal module ordering must be ported bottom-up and
  published to mavenLocal before its dependents can compile.
- **Single-loader-bound APIs** — villager-trade events, custom cauldron interactions, tool-ability
  hooks with no fabric-api equivalent = defer/cut the feature, or the mod stays on the other loader
  (compat layer).

**Rule:** tier every mod at decision time. Budget the Tier-4 mods as multi-session, keep the rest on
the mechanical conveyor, and never let a Tier-0/1 mod accidentally get source-ported.

---

## Appendix — reusable reference points

- **Target vanilla jar** (for signature/descriptor checks) — keep the exact target-version client
  jar on hand; most "does this method still exist / what's its new signature" questions are answered
  by `javap` against it.
- **Decompiler** — Vineflower for clean, Fabric-friendly decompiles of jar-only mods.
- **Mapping scanner** — the intermediary-token grep from §2 (run on *extracted* classes).
- **A small codemod kit** — one re-runnable script per transformation (id-setter, recipe-format,
  rename-table), always measured against a clean compile (§4).
- **A bake-validation script** — resolves blockstate → model → parent → textures across the layered
  resource view; the objective purple/invisible gate (§7).
- **A registry phantom scan** — RCON `execute if block` over every registered id (§7).
- **An in-client placement harness** — a datapack that places every block + summons every mob (§7).
- **A centralized crash-guard mixin mod** — env `"*"`, `targets=`-string mixins, built standalone (§6).
