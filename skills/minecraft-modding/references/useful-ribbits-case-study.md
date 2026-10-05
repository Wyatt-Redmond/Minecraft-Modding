# Case study: porting Useful Ribbits (1.20.1 Forge → 26.2 Fabric/Forge/NeoForge)

A true, blow-by-blow account of taking **Useful Ribbits 1.0.2** (Minecraft 1.20.1, Forge,
MCreator-generated) to **Minecraft 26.2** on **all three loaders** from one shared codebase. This
is the concrete companion to [`porting-playbook.md`](porting-playbook.md) — where the playbook says
"do X," this shows exactly what X looked like for *this* mod, including the dead-ends.

The mod is an **Architectury multiloader** project — `common/` + `fabric/` + `forge/` + `neoforge/` —
on **GeckoLib**, **Java 25**.

---

## 0. Starting point

- **Source:** `useful_ribbits-1.0.2-forge-1.20.1.jar` — 1.20.1 Forge, generated with MCreator (the
  `procedures/` package and names like `RibbitBedGUIRenderChefProcedure` are MCreator's signature),
  license originally unspecified.
- **Content:** three job-ribbit entities (chef / miner / farmer), a **ribbit bed** (GUI that assigns
  a job and previews each ribbit), a **ribbit chest**, three spawn eggs, GeckoLib-animated models.
- **Target:** MC 26.2, Fabric **and** Forge **and** NeoForge, one codebase, GeckoLib 5.5.x, JDK 25.

**Decision-tree result (playbook §1):** no official 26.2 build existed, source was available (the
1.20.1 jar), and the feature set is bespoke → this is a **rung-6 source port**, and a **version lift
of the biggest kind** (playbook §3 job #1: old Forge → new everything — vanilla APIs changed *and*
the loader changed *and* the render pipeline changed). The only external runtime dependency is
**GeckoLib** (the ribbits are GeckoLib entities).

**Why multiloader (Architectury):** rather than port to one loader, the work was structured as a
`common` module (all gameplay, registration, assets) plus three thin loader modules (`fabric`,
`forge`, `neoforge`) that only provide loader-specific glue. This is more up-front structure but
means every gameplay fix lands once and ships to all three loaders.

---

## 1. Build + toolchain (playbook §2)

- **Mojmap, no remap.** 26.2 ships official Mojang names at runtime on every loader, so the build
  uses official mappings with **no `remapJar`** — the Architectury "no-remap" setup. `shadowJar`
  produces the real per-loader jar (common + assets merged); the thin loader jar is not shipped.
- **JDK 25** (26.2 requires it). Bytecode targets a conservative release that still runs on 25.
- **Jar naming:** `UsefulRibbits-26.2-<Loader>-<version>.jar` out of each module's `build/libs`.

The MCreator origin helped here: MCreator code is verbose but mechanical and self-contained (no deep
framework), so it ports largely by codemod + signature fixes rather than architectural surgery.

---

## 2. Registration + platform abstraction (playbook §3)

MCreator's Forge registration (`DeferredRegister`, `@Mod`, Forge events) was replaced with a small
cross-loader layer that lives in `common` and is backed per loader:

- **`DeferredRegistry` / `RegistrySupplier`** — a tiny registry shim so blocks/items/entities/menus
  register declaratively from `common` and resolve on each loader.
- **`IPlatformHelper`** (one interface, three impls) — the only things that genuinely differ per
  loader: a client→server packet send, a `MenuType` factory, the config folder, and (later) a flag
  for the custom bed-preview path. Fabric uses `new MenuType<>(factory, FeatureFlags.VANILLA_SET)`;
  Forge uses `IForgeMenuType.create`; NeoForge uses `IMenuTypeExtension.create`.
- **Networking:** the bed's job buttons send one C2S payload — registered via
  `PayloadTypeRegistry` and sent through the per-loader helper.
- **Item ids at construction:** every item/block now needs its id on `Properties` at construction
  (`new Item(new Item.Properties().setId(key))`), or the ctor throws `Item id not set` — a flat
  codemod across all the MCreator item defs (playbook §4).

---

## 3. The fix log (chronological — the real journey)

Each of these was found by building, running, and hitting it. Order matters: later fixes depended on
the game actually launching.

1. **Fabric wouldn't load at all** — `ServiceLoader … orElseThrow` NPE at startup. The
   `IPlatformHelper` is resolved via `ServiceLoader`, and the `META-INF/services/…IPlatformHelper`
   provider file was missing. Added it for all three loaders. *(Playbook §2: mod-metadata essentials.)*

2. **Spawn eggs were the purple/black missing-texture checkerboard.** The MCreator eggs parented the
   removed `item/template_spawn_egg`. Fixed by **baking authentic tinted egg icons** from the 1.20.1
   grayscale spawn-egg templates × each ribbit's two tint colors, and pointing the models at
   `item/generated`. *(Playbook §7 icon test, §8 purple/invisible.)*

3. **The ribbit bed crashed the moment it opened** — "Tried to access entity ID before ID
   assignment." The bed builds dummy ribbit entities to preview them; in 26.2 those need an explicit
   `setId(...)` before any render-state extraction. Gave each preview entity a stable id.

4. **The ribbit chest had no open animation.** Added a GeckoLib `GeoBlockEntity` + a
   `ribbit_chest.geo.json` / `.animation.json`, set the block's `getRenderShape → INVISIBLE`, and
   rendered it through a GeckoLib BER with a `triggerAnim` open animation. (Several iterations on the
   hinge pivot and UVs — GeckoLib UVs are in texel space and block models must be X/Z-centered.)

5. **Chef ribbits all crowded a single smoker.** The AI goal picked the nearest smoker for everyone.
   Rewrote the smoker search to **distribute** ribbits across all nearby smokers, **prefer smokers
   that already have fuel**, and **migrate** to a newly placed smoker — plus a stuck-watchdog.

6. **Farmer ribbits stopped replanting** after a crop was broken. Rewrote the chest-wait handler to
   **replant from the seeds they're already holding** instead of idling.

7. **Ribbits "got stuck" in general.** Tightened the job-scanning throttle and added stuck-watchdogs
   across the goals so they re-scan for work instead of parking.

8. **Croak frequency config.** Added a small Gson-POJO config (`UsefulRibbitsConfig` +
   `JsonIO`) in the config folder — ambient-croak toggle + interval. *(Playbook §3: config = plain
   Gson POJO, no config-lib dependency.)*

9. **Cross-mod recipe compatibility.** The bed recipe wanted a Ribbits "toadstool." To work whether
   the player has this port **or** the official Ribbits mod, the recipe uses an item tag with an
   **optional** entry: `#useful_ribbits:bed_base = #minecraft:wool + {"id":"ribbits:toadstool",
   "required":false}`.

10. **The hard one — the Forge-only bed-preview collapse.** On **Forge only**, the bed's three job
    previews all rendered as the **last** ribbit (the farmer); Fabric and NeoForge were fine with
    identical code. See the dedicated writeup in the next section — this is the most instructive bug
    in the whole port.

11. **A full code audit** (a parallel review pass) found and fixed ~17 smaller issues across the
    goals, registration, and client code.

---

## 4. Deep-dive: the Forge bed-preview collapse (playbook §5, "picture-in-picture previews")

This one is worth reading in full because it looks impossible ("same code, works on 2 of 3 loaders")
until you understand 26.2's retained-mode GUI.

**Symptom:** open the ribbit bed on Forge → all three preview slots show the farmer. On Fabric and
NeoForge the same build shows chef / miner / farmer correctly. In-world, all three ribbits render
correctly on every loader. So it's *specifically* the GUI preview, *specifically* on Forge.

**What was ruled out (by decompiling GeckoLib + both loaders' Minecraft jars and byte-comparing):**
- Our code is symmetric — three distinct entity types, renderers, GeckoLib models, geo identifiers,
  textures, and cached preview entities with distinct ids.
- GeckoLib builds a **fresh** render state every call (`new LivingEntityRenderState()`); there's no
  shared/reused state, and its per-state data map is a real per-instance field. All of this is
  **byte-identical** across the three loaders.
- The vanilla GUI entity path (`extractEntityInInventoryFollowsMouse` → `GuiGraphicsExtractor.entity`
  → `GuiEntityRenderer` → `EntityRenderDispatcher.submit`, which resolves the renderer from
  `state.entityType`) is **byte-identical between vanilla(Fabric) and Forge-patched** Minecraft.

**Actual root cause (26.2 retained-mode mechanics):** every entity preview goes through the single
shared `GuiEntityRenderer`. The pip framework renders each preview into that renderer's **one reused
GPU texture** and records a **deferred blit** of that texture. `GuiRenderer.render()` does
`prepare()` (renders *all* pips first → the shared texture ends up holding the last ribbit) and only
then `draw()` (executes *all* the deferred blits → every blit reads that one texture = the farmer).
Three previews in one frame therefore collapse to the last. Fabric/NeoForge tolerate this at runtime;
Forge 26.2 does not.

**Fix:** give each slot its **own** picture-in-picture renderer, so no renderer ever renders more
than one preview per frame (and thus never reuses its texture mid-frame). Concretely, on Forge:
- A `RibbitPreviewState` (implements `PictureInPictureRenderState`, incl. `bounds()`) with three
  trivial subclasses — one per slot, so each dispatches to its own renderer by class.
- A `RibbitPreviewPipRenderer` that reproduces vanilla `GuiEntityRenderer` exactly (`getTranslateY`,
  `renderToTexture`), so previews look pixel-identical.
- Register the three via Forge's `RegisterPictureInPictureRendererEvent`, and submit them from the
  screen (reflecting the extractor's `guiRenderState` — safe because 26.2 ships unobfuscated).
- Fabric and NeoForge keep the stock `extractEntityInInventoryFollowsMouse` path (it already works),
  gated by a one-line `IPlatformHelper.useCustomEntityPreview()` that's true only on Forge.

**The lesson for a 26.2 author (playbook §5):** if you render an entity in a GUI slot, the
retained-mode pip path shares one texture across same-type previews within a frame — fine for one
preview, a trap for several. Either render one entity per frame or give each its own pip renderer.

**A related self-inflicted wound worth flagging:** mid-debugging, an interim attempt called the
no-arg `renderer.createRenderState()` — which GeckoLib stubs to return **null** — and crashed. The
correct state factory is `EntityRenderDispatcher.extractEntity(entity, partialTick)`. And when the
Forge fix was deployed but Fabric/NeoForge weren't rebuilt, they crashed on the **stale** interim
jar — a reminder that a shared-`common` change means *rebuild every loader*, not just the one you
touched.

---

## 5. Validation (playbook §7)

- Clean compile on all three modules.
- Built all three loader jars green.
- **Runtime-verified on all three loaders**: the mod loads, the three ribbits spawn with their
  correct models, the bed opens and shows three distinct previews (the whole point), the chest
  animates, the spawn eggs show their real icons, and the jobs (cooking, farming) work.

The GUI preview bug is the case in point for the playbook's rule that *boot-to-Done proves nothing
about client rendering* — it only surfaced by opening the screen in a real client on each loader.

---

## 6. What a greenfield rewrite would change

Since the plan is to rebuild Useful Ribbits from scratch, cleaner and more scalable, a few things
this port carries would simply not exist in fresh code — see
[`writing-from-scratch.md`](writing-from-scratch.md) for the full treatment:

- **No MCreator `procedures/`** — the job logic becomes real `Goal`s and block/menu classes, not
  generated procedure shells.
- **Data components over NBT** for item state, and the modern registration/`setId` model from day one.
- **The retained-mode GUI and the pip-preview pattern designed in**, not retrofitted — the bed's
  per-slot preview would be built the per-renderer way from the start, so there's no Forge special
  case.
- **One deliberate platform-abstraction seam** (`IPlatformHelper`-style) instead of discovering it
  fix by fix.

---

*This port is a community 26.2 continuation published with the original author's blessing. Original
Useful Ribbits by rogue_one (Rogue_one12); base ribbit assets © Refresh Studios & Bonsai Studios.*
