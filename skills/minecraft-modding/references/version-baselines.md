# Minecraft 26.x modding baselines (the version matrix)

The build/runtime baseline every 26.x mod must match: Java, pack formats, mapping model, and the
loader / API / toolchain versions each release expects. This is the "what does Minecraft expect for
26.x+" lookup — pair it with the `javap` delta method in
[version-bump-method.md](version-bump-method.md) when you need a version's *API* changes, and with
[official-docs-and-primers.md](official-docs-and-primers.md) for the canonical per-loader toolchain,
the Yarn-EOL/mapping facts, and the NeoForge primer chain (the authoritative source for the semantic
vanilla deltas `javap` can't reveal). Note: **official Forge docs stop at 1.21.x** — the 26.x Forge
versions below come from the Forge maven/files site and are unofficial.

**Verified 2026-10-05** against live sources (Mojang piston-meta, misode/mcmeta, Fabric meta,
Modrinth, NeoForge/Forge mavens). **These numbers drift — re-verify before trusting a bump** (how +
the exact endpoints are at the bottom; a carried-over number is exactly how the stale `34` bug below
happened). Scope: stable releases **26.1 → 26.3**. **26.4 is snapshots-only as of 2026-10-05** (not
stable, not baselined).

## Baseline matrix

| Version | Java | Resource pack fmt | Data pack fmt | Mappings | Fabric loader | Fabric API | NeoForge | Forge | Loom (no-remap) | GeckoLib |
|---|---|---|---|---|---|---|---|---|---|---|
| 26.1 | 25 | 84 | 101 | mojmap (no remap) | 0.19.5 | 0.155.3+26.1.2 | 26.1.0.19-beta † | 26.1-62.0.9 | 1.13.x ‡ | 5.5 |
| 26.1.1 | 25 | 84 | 101 | mojmap (no remap) | 0.19.5 | 0.155.3+26.1.2 | 26.1.1.15-beta † | 26.1.1-63.0.2 | 1.14.x ‡ | 5.5 / 5.5.1 § |
| 26.1.2 | 25 | 84 | 101 | mojmap (no remap) | 0.19.5 | 0.155.3+26.1.2 | **26.1.2.114** | 26.1.2-64.1.3 | 1.14.x ‡ | 5.5.2 |
| 26.2 | 25 | **88** | 107 | mojmap (no remap) | 0.19.5 | 0.161.0+26.2 | **26.2.0.88** | 26.2-65.1.3 | 1.17.x ‡ | 5.5.5 |
| 26.3 | 25 | **97** | **121** | mojmap (no remap) | 0.19.5 | 0.161.0+26.3 | 26.3.0.51-beta † | 26.3-66.0.9 | **1.17.491** | 5.5.7 |

- **†** NeoForge is **beta-only** for that MC version (no stable). First stable NeoForge in the 26.1
  line is **26.1.2.114** → target MC **26.1.2** for a stable NeoForge. 26.3 NeoForge is still beta.
  NeoForge versioning: `26.A.B.C[-beta]` targets MC `26.A.B` (`C` = build); there is no promotions
  file, so stable-vs-beta is read from the `-beta`/`-alpha` suffix only.
- **‡** Loom build is **date-correlated/approx**, not anchored to a shipped port. The toolchain is
  **generation-wide**: latest **Architectury Loom 1.17.x + architectury-plugin 3.5.x + Gradle ≥9.1**
  builds **any** 26.x — select the MC version in `gradle.properties`, not via the tool version. Only
  26.3's **Loom 1.17.491 + plugin 3.5.169** are hard-anchored to a shipped build.
- **§** GeckoLib has **no build tagged 26.1.1** — use 5.5 (26.1) or 5.5.1 (26.1.2); patch-line
  binary-compatible. Also: **5.5.5 is the last GeckoLib with a Fabric jar for 26.2** (5.5.6 for 26.2
  is forge+neoforge only).

**Build-time toolchain (all 26.x, kept out of the table for width):** architectury-plugin 3.5.x
(`3.5.169` anchored for 26.2/26.3); Gradle 9.x, floor **9.1** (first Gradle with the Java 25
toolchain); Forge `-recommended` promos exist only for `26.1.2-64.1.0` and `26.2-65.1.0` (26.1 /
26.1.1 / 26.3 are latest-only).

## Pack formats — resource and data now DIVERGE ⚠

Since 26.x there is **no single `pack_format`**: the resource (assets) format and the data format are
**different integers** and drift apart. A `pack.mcmeta` uses the **resource** number for assets and
the **data** number for data. Verified: 26.1.x = res **84** / data **101**; 26.2 = res **88** / data
**107**; 26.3 = res **97** / data **121**.

- **The `34` trap:** `34` is the **1.21 / 1.21.1 resource format** — a stale value that gets carried
  into a 26.x `pack.mcmeta` and never bumped. It must **never** appear in a 26.x pack. (This is a real
  bug we shipped: our 26.3 Ribbits ports carried `34` forward; correct is `97`/`121`.)
- `88` (our 26.2 value) was **correct** — it is the 26.2 resource format.
- Never reuse one number for the other, and prefer the new split `min_format`/`max_format`
  (`[major, minor]`) over a lone legacy `pack_format`, which can get a standalone pack rejected
  (black screen). Mod jars are more lenient than standalone packs, but set the right number anyway.

## True for every 26.x (generation-wide)

- **Mappings = mojmap / unobfuscated, no remap, no intermediary, no yarn** — on *all* loaders. Proven,
  not assumed: every 26.x piston-meta manifest exposes only `client`/`server` downloads (no
  `client_mappings`, unlike obfuscated 1.21.x) and Fabric meta returns **0 yarn builds** for every
  26.x. Do **not** add a Loom `remapJar`/deobf step.
- **Java 25** everywhere (`javaVersion.majorVersion = 25`, `java-runtime-epsilon`). JDK 25 toolchain;
  Gradle ≥ 9.1. Cross-era ladder: 1.20.2–1.20.4 = Java 17, 1.20.5–1.21.11 = Java 21, **26.x = Java 25**
  (bumped at the 1.21.11→26.1 boundary). ⚠ NeoForge's *user* install table still reads "Java 21" for
  current — stale; the 26.1 primer's Java 25 is authoritative.
- **Ship a shadow/fat jar, not a thin jar** — bundle (and relocate where needed) your deps via
  `shadowJar` + Architectury `shadowBundle`; a thin jar fails at runtime in a modpack.
- **LWJGL build-pin:** MC ships LWJGL **3.4.1** on 26.1/26.1.1/26.1.2/26.2 (no action) and **3.4.3**
  on **26.3** (and presumably later). The Architectury transformer's bundled ASM can't read the
  Java-27-compiled 3.4.3 classfiles, so on any version shipping 3.4.3 **force the build-time LWJGL dep
  to 3.4.1** (runtime still loads MC's own 3.4.3). 3.4.3 also swapped glfw→sdl and added the Vulkan
  modules.
- **Version naming:** Forge = `<mc>-<forge>` (forge major bumps per MC: 26.1→62.x … 26.3→66.x);
  Fabric loader is version-agnostic (same 0.19.5 across all 26.x); Fabric API is tagged `+<mc>` and
  one build can cover a whole patch line (`0.155.3+26.1.2` covers 26.1/26.1.1/26.1.2).

## Verify before you trust a bump

A pack.mcmeta number or a loader version carried from the last release is how things rot. Before any
26.x bump, re-fetch from source:
- **Pack formats** → `raw.githubusercontent.com/misode/mcmeta/summary/versions/data.min.json`
- **Java major + LWJGL** → Mojang `launchermeta.mojang.com/.../version_manifest_v2.json` → the
  version's piston-meta URL (`javaVersion.majorVersion`; grep `org.lwjgl:lwjgl`)
- **Fabric loader/API** → `meta.fabricmc.net/v2/versions/loader` + `api.modrinth.com/v2/project/fabric-api/version`
- **NeoForge** → `maven.neoforged.net/releases/net/neoforged/neoforge/maven-metadata.xml`
- **Forge** → `files.minecraftforge.net/net/minecraftforge/forge/promotions_slim.json` + maven-metadata
- **GeckoLib** → `api.modrinth.com/v2/project/geckolib/version`

**Rule:** match the whole baseline row for your target version, derive the API delta with the `javap`
method, and re-verify these numbers from source — never trust a carried-over `pack.mcmeta`.
