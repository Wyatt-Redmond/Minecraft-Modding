# Writing a Minecraft 26.2 mod from scratch (clean, multiloader, no MCreator)

A companion to the [porting playbook](porting-playbook.md) and the
[Useful Ribbits case study](useful-ribbits-case-study.md), aimed at a **greenfield rewrite** —
building a mod cleanly for 26.2 on Fabric/Forge/NeoForge from one codebase, by someone comfortable
with programming but newer to Java/Minecraft modding. The porting docs tell you how 26.2 *differs
from the past*; this one tells you how to *start correctly* so you never accumulate that debt.

Opinionated on purpose. These are defaults that scale; deviate when you have a reason.

> **These defaults are the 26.2 era.** The project-layout and platform-seam structure (§1–2) is
> version-agnostic, but the API/registration/JSON/render specifics (§3–6) are a snapshot — for any
> other target (a future 27, or an older 1.16.5), confirm each against your version with the `javap`
> method in [version-bump-method.md](version-bump-method.md), the per-version numbers in
> [version-baselines.md](version-baselines.md), and the official toolchain/primer facts in
> [official-docs-and-primers.md](official-docs-and-primers.md); the era map is in the skill root.

---

## 1. Project layout — one codebase, three loaders (Architectury)

Split "what the mod does" from "how each loader wires it up":

```
mymod/
├─ settings.gradle            # include 'common','fabric','forge','neoforge'
├─ gradle.properties          # versions in ONE place (mc, loader, api, geckolib…)
├─ build.gradle               # shared Loom/Java config for all subprojects
├─ common/                    # ALL gameplay, registration, assets, data. No loader imports.
│   └─ src/main/{java,resources}
├─ fabric/                    # entrypoints + platform impl only
├─ forge/
└─ neoforge/
```

- **`common` holds everything real** — blocks, items, entities, menus, AI, and *all* assets/data
  (`assets/`, `data/`). It must not import `net.minecraftforge.*` / `net.neoforged.*` /
  `net.fabricmc.*`. Anything loader-specific goes behind an interface (see §2).
- **Each loader module is thin** — a mod entrypoint that calls into `common`, plus the
  implementation of your platform interface. If a loader module grows gameplay logic, it belongs in
  `common` instead.
- **Build model (26.2): mojmap, no remap.** 26.2 ships official Mojang names at runtime on every
  loader, so compile against official mappings with **no intermediary `remapJar` step**. The
  complete per-loader jar is the **shadowJar** output (common + assets merged), not the thin loader
  jar. Use the **JDK the target MC version requires** (26.2 → JDK 25; verify for a newer target).
  **Yarn is EOL** from the first unobfuscated MC onward — a new 26.x Fabric project carries **no
  `yarn_mappings` key**; to port an *existing* Yarn mod, use Loom's `migrateMappings` task (no Kotlin)
  or the Ravel IntelliJ plugin, not a hand rename.

**Why this over single-loader:** more up-front scaffolding, but every feature is written once and
ships to all three loaders, and the loader differences are quarantined to a handful of small classes.

---

## 2. The platform seam (do this once, early)

The only things that genuinely differ per loader are a short list: client→server networking, menu
(`MenuType`) creation, the config directory, loader/env checks, and a few client register hooks.
Put them behind **one interface in `common`**, resolve it per loader with `ServiceLoader`:

```java
// common
public interface IPlatformHelper {
    <T extends CustomPacketPayload> void sendToServer(T payload);
    <T extends AbstractContainerMenu> MenuType<T> menuType(MenuType.MenuSupplier<T> factory);
    Path getConfigFolder();
}
```

- Each loader module ships a `META-INF/services/<...>.IPlatformHelper` provider file naming its impl.
  **Forgetting that file = a `ServiceLoader … orElseThrow` crash at startup** (the single most common
  first-boot failure).
- Fabric `MenuType`: `new MenuType<>(factory, FeatureFlags.VANILLA_SET)`. Forge:
  `IForgeMenuType.create(...)`. NeoForge: `IMenuTypeExtension.create(...)`.
- Resolve once: `ServiceLoader.load(IPlatformHelper.class).findFirst().orElseThrow()`.

Keep this interface small. If it grows past ~6 methods, you're probably leaking gameplay into it.

---

## 3. Registration the modern way

26.2 registration has two hard rules that bite every new mod:

1. **An object's id must be on its `Properties` at construction.**
   `new Item(new Item.Properties().setId(ResourceKey.create(Registries.ITEM, id)))` — a bare
   `new Item(new Item.Properties())` throws `NullPointerException: Item id not set`. Blocks use
   `Registries.BLOCK`.
2. **`Holder` is sealed in vanilla.** You cannot write your own class that `implements Holder`
   (Forge/NeoForge un-seal it; Fabric does not). If you build a registration helper, expose `value()`/`get()`
   instead of pretending to be a `Holder`.

Two clean options:

- **Vanilla registry calls** (simplest, fully cross-loader):
  `Registry.register(BuiltInRegistries.ITEM, id, new Item(props.setId(key)))`. Wrap them in a tiny
  `DeferredRegistry`/`RegistrySupplier` helper in `common` so call sites stay declarative and you
  register in a deterministic order.
- A **register-time `ThreadLocal<ArrayDeque<ResourceKey>>` + `Properties`-ctor mixin** that stamps
  `setId` automatically — only worth it if you have *hundreds* of call sites. It must be a per-thread
  **stack** (ArrayDeque), because nested registration (a block's Properties touching a sound constant
  mid-construction) will corrupt a single-slot holder. For a fresh, modestly sized mod, prefer the
  explicit `setId` — it's less magic.

**Never construct an `ItemStack` / call `getDefaultInstance()` in a `<clinit>`** of any class that
can load during resource reload (block-entity renderers, entity renderers, models, particles). Data
components aren't bound yet at that point → `IllegalStateException: Components not bound yet`. Make
such stacks lazy (`Supplier<ItemStack>`).

---

## 4. Items are data components now (not NBT, not subclasses)

The old "subclass `Item`/`ArmorItem`/`RecordItem` and override behavior" model is largely gone;
behavior is attached as **data components** on `Item.Properties`:

- Edible → `.component(DataComponents.CONSUMABLE, Consumable.builder()...build())` (`.food(...)` alone
  no longer drives eating in 26.2).
- Equipable → `EQUIPPABLE` component (not `ArmorItem`). Music disc → `JUKEBOX_PLAYABLE` +
  a `JukeboxSong` datapack entry (not `RecordItem`). Spawn egg → `Item.Properties.spawnEgg(type)`
  with the ctor taking only `(Properties)`; the two colors are a client tint.
- Per-stack state → store it in a **custom `DataComponentType`**, not raw NBT. Register your component
  type like any other registry object.

For block/entity **save data**, 26.2 uses `ValueInput`/`ValueOutput` (codec-backed), not
`CompoundTag`: `loadAdditional/saveAdditional(ValueInput/ValueOutput)` and
`readAdditionalSaveData(ValueInput)`. UUIDs/BlockPos go through codecs
(`store/read(key, UUIDUtil.CODEC)` / `BlockPos.CODEC`).

---

## 5. Everything data-driven ships as JSON — and 26.2 is picky

- **Item models need TWO files.** `assets/<ns>/models/item/foo.json` (the geometry) **and**
  `assets/<ns>/items/foo.json` (the item *definition*:
  `{"model":{"type":"minecraft:model","model":"<ns>:item/foo"}}`). Miss the `items/` def and the icon
  is purple. This is the #1 "my icons are missing" cause.
- **Recipes use the flat ingredient format:** `"minecraft:stick"` (not `{"item":"minecraft:stick"}`),
  `"#tag"` (not `{"tag":"tag"}`), and a JSON array for multiple options. Result stays
  `{"id":"...","count":N}`.
- **`pack.mcmeta` needs `min_format`/`max_format`** ([major,minor] arrays) — legacy
  `pack_format`/`supported_formats` alone can get the pack rejected (black screen).
- **Cross-mod soft deps in recipes:** reference another mod's item through a tag with an optional
  entry — `{"id":"othermod:thing","required":false}` — so the recipe works whether or not that mod
  is present.
- Prefer **datagen** for anything repetitive (models, blockstates, loot, recipes), but remember
  datagen classes never run at runtime — don't let runtime code depend on them.

---

## 6. Rendering: 26.2 is retained-mode (extract → render)

This is the biggest conceptual shift from older versions, and designing for it up front saves the
most pain.

- **GUIs don't draw; they extract.** `GuiGraphics` is gone. Screens/widgets override `extract*`
  methods and write render state into a `GuiGraphicsExtractor`; `GuiRenderer` renders it later.
  `AbstractContainerScreen`: `renderBg → extractBackground`, `renderLabels → extractLabels`, etc.
  Drawing calls: `g.text(...)`, `g.blit(RenderPipelines.GUI_TEXTURED, id, ...)`, `g.item(stack,x,y)`.
  `g.pose()` is a **2D** `Matrix3x2fStack` (no z). Tooltips are deferred:
  `setTooltipForNextFrame(...)`.
- **In-world custom geometry uses the submit pipeline.** `MultiBufferSource` is gone. A
  block-entity/entity renderer is `createRenderState()` → `extractRenderState(...)` →
  `submit(state, PoseStack, SubmitNodeCollector, camera)`. Template on vanilla `BeaconRenderer`.
  Always wrap `submit`/`extractRenderState` bodies in try/catch — the dispatcher rethrows as a hard
  crash; degrade to "not drawn + one log line."
- **Rendering an entity in a GUI slot (e.g. a mob preview):** the retained-mode "picture-in-picture"
  path renders all same-type previews through one shared renderer into one reused texture with a
  **deferred** blit. One preview is fine; **several in one frame collapse to the last one on Forge**.
  If you need multiple live previews side by side, give each its **own** pip renderer (own texture) —
  see the case study §4 for the exact pattern. Design it that way from the start and there's no
  loader special-case.
- **Animated models:** **GeckoLib** is the standard. A `GeoEntity`/`GeoBlockEntity` + `GeoModel`
  (bedrock `.geo.json` + `.animation.json`) + a `GeoEntityRenderer`/`GeoBlockRenderer`. Register
  renderers per loader (Fabric client init / Forge+NeoForge `EntityRenderersEvent`). GeckoLib builds
  a fresh render state per call and stores its data on the state — play nicely with that and it's
  smooth.

---

## 7. Networking, config, events

- **Networking:** define a `CustomPacketPayload` with a `StreamCodec`, register it with
  `PayloadTypeRegistry.clientboundPlay()/serverboundPlay()`, and send it through your platform helper
  (Fabric `ServerPlayNetworking.send` / client distributor on NeoForge, etc.). Validate inputs on the
  **server** side of any C2S packet — never trust the client.
- **Config:** a plain **Gson POJO** read/written in `FabricLoader.getConfigDir()` (Gson is already on
  Minecraft's classpath). No config-library dependency needed for something simple, and it's trivially
  cross-loader.
- **Events/callbacks:** on Fabric these are the fabric-api callbacks (`ServerPlayerEvents.COPY_FROM`,
  `ItemTooltipCallback.EVENT`, `CommandRegistrationCallback.EVENT`, `HudElementRegistry`, …). On
  Forge/NeoForge they're the loader event buses. Keep the handler logic in `common` and just register
  it per loader.

---

## 8. Clean-code / scalability notes

Targeted at keeping a mod maintainable as it grows (and at avoiding the traps that forced special
cases in the port):

- **Quarantine loader differences** behind the platform interface; keep `common` loader-agnostic. The
  moment you reach for a loader import in gameplay code, stop and add a method to the seam instead.
- **Register in deterministic order**, and keep registry objects as `static final` suppliers grouped
  by type (`ModItems`, `ModBlocks`, `ModEntities`, `ModMenus`) — easy to read, easy to datagen from.
- **Lazy over eager** for anything touching components/registries at class-load (§3).
- **Prefer data over code** — a new block/recipe/loot entry should mostly be JSON, not a new class.
- **Validate the client, not just the server.** A dedicated-server boot never exercises GUI/render
  code; open every screen and spawn every entity on a real client before calling it done (the GUI
  preview bug in the port only showed up in-client, per-loader).
- **A shared-`common` change means rebuild *every* loader**, not just the one you tested — mixed
  stale jars crash in confusing ways.
- **Rebuild with the game closed.** Rewriting a jar while Minecraft is running corrupts it
  (`ZipException: bad LOC`) and looks like a crash that isn't one.

---

## 9. Minimal "it builds and loads" checklist

1. `gradle.properties` versions set; `common` + three loader modules; mojmap no-remap; JDK 25.
2. `IPlatformHelper` + a `META-INF/services` provider file in **each** loader module.
3. One block or item registered with `setId` on its `Properties`; its `models/item` **and** `items/`
   defs present.
4. Entrypoints wired (Fabric `fabric.mod.json`, Forge `mods.toml`, NeoForge `neoforge.mods.toml`);
   `minecraft` dependency predicate widened (e.g. `">=26.1 <27"` — the upper bound must sit *above*
   your target; `<27` would exclude MC 27), depend on `fabric-api` (not the legacy `fabric` alias).
5. `./gradlew build` → three shadowJars.
6. Boot a dedicated server to `Done` (validates common), **then** open the client and exercise the UI
   (validates render).

Get that loop green first, then grow the mod one feature at a time against it.

---

*Written as a starting-point guide for a clean 26.2 rebuild. Pair it with the porting playbook for the
exhaustive API-delta detail.*
