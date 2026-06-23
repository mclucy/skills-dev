---
name: minecraft-ecosystem
description: Use when designing, implementing, or reviewing features that interact with the Minecraft modding and plugin ecosystem — especially package identity, version resolution, platform detection, dependency graphs, or multi-loader support. Also use when assumptions about mod loader behavior, metadata formats, or registry conventions seem too clean to be true.
---

# Minecraft Modding Ecosystem

## Overview

The Minecraft modding ecosystem is fragmented by design. No central authority governs naming, versioning, metadata formats, or dependency declarations across mod loaders and plugin systems. Any tool that attempts to unify these systems must treat inconsistency as the default.

**This skill is a routing map, not an encyclopedia.** It points at the authoritative source for each topic and names the assumptions most likely to break. Verify before you commit to an abstraction; do not memorize field lists — they will go stale.

**Cardinal sin:** assuming that any two platforms, registries, or metadata formats agree on identity, versioning, or semantics. They don't. Every cross-cutting abstraction must answer the question "what happens when this assumption breaks?"

## When to Use

- Designing identity, naming, or scoping systems for Minecraft packages
- Implementing version resolution across mod loaders or registries
- Parsing or interpreting artifact metadata (JARs, plugin descriptors, Python packages)
- Building platform/runtime detection that must distinguish server cores, mod loaders, and plugin frameworks
- Adding support for a new platform, registry, or server core
- Reviewing code that makes assumptions about how the Minecraft ecosystem behaves

## Quick Reference: Authoritative Sources per Platform

Each row names the metadata file, the identity field, and **where to verify the rules**. Do not paraphrase from memory — follow the link.

| Platform | Metadata File | Identity Field | Verify rules at |
|---|---|---|---|
| Fabric | `fabric.mod.json` | `id` | [fabric-docs: fabric-mod-json](https://docs.fabricmc.net/develop/loader/fabric-mod-json) — also `fabric-loader/.../metadata/MetadataVerifier.java`. Regex `[a-z][a-z0-9-_]{1,63}`. |
| Forge (1.13+) | `META-INF/mods.toml` | `modId` | NeoForge `fmlcore/.../ModInfo.java`. Regex `^[a-z][a-z0-9_]{1,63}$` (underscore only, **no hyphens**). |
| Forge (≤1.12) | `mcmod.info` | `modid` (lowercase) | Legacy. Array-based JSON. Modern Forge detects only to reject. |
| NeoForge | `META-INF/mods.toml` | `modId` | Same parser as modern Forge. `META-INF/neoforge.mods.toml` is **only** for NeoForge's own bundled mods — third-party mods use `mods.toml`. |
| Bukkit / Spigot / Paper / Folia / Leaves | `plugin.yml` | `name` (NOT `id`) | `bukkit/.../PluginDescriptionFile.java`. Pattern `^[A-Za-z0-9 _.-]+$`, spaces normalized to `_`. |
| BungeeCord / Waterfall | `plugin.yml` (yes, same filename) | `name` | BungeeCord `ModuleManager.java` reads `plugin.yml`, not `bungee.yml` despite the convention. |
| Velocity | `velocity-plugin.json` | `id` | Velocity `JavaPluginLoader.java`. Regex `[a-z][a-z0-9-_]{0,63}`. Velocity rejects `plugin.yml`/`bungee.yml`/`paper-plugin.yml`. |
| Sponge | `META-INF/sponge_plugins.json` | `plugins[].id` | Sponge `PluginPlatformConstants.java` and `PluginFileParser.java`. Kebab-case. Multiple plugins per JAR. |
| MCDR | `mcdreforged.plugin.json` | `id` | `MCDReforged/.../metadata.py` — runtime regex `[a-z][a-z0-9_]{0,63}`. The JSON **schema** permits hyphens but the Python parser does not; schema is stale, trust runtime. |

## Rule: Cite the Source for Domain-Dependent Code

When you write code that depends on external behavior or specific Minecraft-domain knowledge, **append the exact documentation URL in a comment** right next to the code that implements it. Keep the comment brief — one line naming the assumption and the link that backs it.

What counts as "domain-dependent":
- A regex or parser derived from a platform's metadata rules (e.g., Fabric's `id` regex, Forge's `modId` regex).
- A hash algorithm or endpoint shape dictated by a registry (e.g., CurseForge's murmur2 fingerprint, Modrinth's `/v2/version_file/{hash}`).
- A detection heuristic based on a runtime signal that is not in any spec (e.g., "Folia advertises via `folia-supported`, not via Paper").
- A version-range parser matching a platform's dialect (Fabric semver, Maven ranges, MCDR custom).
- Any rule that starts with "Minecraft platforms do X" or "Forge expects Y".

Good comment shape:
```go
// Fabric mod IDs are lowercased, 2-64 chars, allow hyphens and underscores.
// Source: https://github.com/FabricMC/fabric-loader/blob/.../MetadataVerifier.java
var fabricIDRe = regexp.MustCompile(`^[a-z][a-z0-9-_]{1,63}$`)
```

Why: domain rules change without notice across versions, and a future reader cannot tell whether a regex or magic string came from a spec or was invented. The URL turns "why does this look like this?" into a one-click answer. Prefer a deep link to the specific source file or doc section over a landing page.

When in doubt, cite. If you cannot find a source URL to cite, that is a signal the assumption is unverified — see §1 of Key Inconsistencies.

## Key Inconsistencies (Each Is a Pointer, Not a Treatise)

Each inconsistency below is a load-bearing assumption that breaks in production. Verify the current state against the linked source before relying on it.

### 1. Local IDs Don't Match Remote Slugs

**The cardinal case.** A mod's local ID and its registry slug are independently chosen by different parties and may diverge, be renamed, or be reused by unrelated projects.

**REQUIRED:** See [`package-names.md`](package-names.md) for the structural reasons, per-platform/per-registry verification URLs, lucy's bridging strategy (slugmap + hash lookup), and signals your mapping assumption is wrong.

### 2. Local Identity Uniqueness Is Per-Server Only

A mod loader enforces uniqueness within one running server. There is no global registry for local IDs — two unrelated mods on different servers can share the same `id`.

**Meta-cognition:** Local identity is a runtime constraint, not a namespace guarantee. Don't treat "this ID" as globally addressable.

### 3. Normalization Is Reader-Defined, Not Spec-Defined

Some fields get normalized (lowercased, underscores→hyphens, spaces→underscores). Whether normalization happens depends on the *reader implementation*, not on the platform's spec.

**Meta-cognition:** Always define where normalization happens in your pipeline. Don't assume upstream metadata is already normalized; don't normalize twice. Bukkit `name` keeps case and spaces; CurseForge slugs don't. The seam between them is yours to handle.

### 4. Multiple Mods Per Artifact

Forge, NeoForge, and Sponge permit multiple mod/plugin entries in a single JAR. A single downloaded artifact can produce multiple identities.

**Meta-cognition:** The artifact→identity relationship is one-to-many. Systems that index by identity must iterate; systems that index by file must allow multiple hits.

### 5. Version Placeholders and Version-Scheme Mismatch

Forge/NeoForge mods can declare `version = "${file.jarVersion}"` in `mods.toml`, resolved at load time from `META-INF/MANIFEST.MF → Implementation-Version`. If the manifest entry is missing, the version is empty. Separately, the *version range* syntax differs per platform:

- **Fabric:** semver with `~`/`^` operators (predicate arrays, NOT Maven ranges).
- **Forge / NeoForge:** Maven ranges — `[1.0,2.0)`, `(,1.0]`, `[1.0]`, `[1.0,)`.
- **MCDR:** custom space-separated predicates with `^`/`~` — neither semver nor Maven.
- **Minecraft itself:** release (`1.21.4`) and snapshot (`25w14a`) schemes that are neither.

**Verify at:** `lucy/version/version_range_dialect.go` — lucy dispatches by platform. The authoritative parsers are `fabric-loader/.../VersionPredicateParser.java`, NeoForge's `MavenVersionAdapter`, and `MCDReforged/.../version.py`.

**Meta-cognition:** "Parse the version" is three different problems depending on platform. Don't write one regex.

### 6. Bukkit-Family Platform Detection Is Heuristic

There is no authoritative "I am Paper" / "I am Folia" field in `plugin.yml`. The exact Bukkit-family platform (Bukkit, Spigot, Paper, Folia, Leaves, Purpur, Leaf) is inferred from multiple weak signals: `api-version`, `folia-supported` (Folia-only, refuse-without), paper-plugin.yml's `bootstrapper`/`loader`, strings in `depend`/`softdepend`/`libraries`, and class presence (`com.destroystokyo.paper.*`, `io.papermc.paper.*`) on the server classpath.

**Verify at:** `lucy/artifact/reader_bukkit.go::detectBukkitPluginPlatform` — infers `leaves > folia > paper > spigot > bukkit` from a 6-stage pipeline. Also `lucy/workspace/detector/` for the runtime classpath side.

**Meta-cognition:** Detection is probabilistic. Design for "platform unknown" as a first-class outcome. Prefer capability checks over identity checks.

### 6a. The Bukkit Inheritance Ladder (Load-Bearing)

The Bukkit family is a strict inheritance ladder. Each rung extends the API surface of the rung below it, and **forward compatibility flows downward**: a higher-rung server can host plugins built for any lower rung, but a plugin using higher-rung APIs will fail on a lower-rung server (missing classes at load time).

```
Vanilla Minecraft
  └─ CraftBukkit (implements Bukkit API)
       └─ Spigot (extends CraftBukkit, adds Spigot API)
            └─ Paper (extends Spigot, adds Paper API — modern baseline, ~85-90% market share)
                 ├─ Purpur (extends Paper, adds Purpur API)
                 │    └─ Leaves, Leaf (Purpur forks; same plugin ecosystem, no separate capability)
                 └─ Folia (Paper fork, INVERTS compatibility — see below)
```

**API surface per rung:**

| Rung | API packages | Capability constant |
|---|---|---|
| Bukkit | `org.bukkit.*` | `CapabilityBukkitPlugins` |
| Spigot | + `org.spigotmc.*` | `CapabilitySpigotPlugins` |
| Paper | + `io.papermc.paper.*` (+ legacy `com.destroystokyo.paper.*`) | `CapabilityPaperPlugins` |
| Purpur | + `org.purpurmc.purpur.*` | `CapabilityPurpurPlugins` |
| Folia | + `io.papermc.paper.threadedregions.scheduler.*` | `CapabilityFoliaPlugins` |

**Compatibility direction:**
- **Normal ladder (Bukkit → Spigot → Paper → Purpur):** Forward-compatible. Paper runs Spigot plugins; Spigot runs Bukkit plugins. Reverse is NOT true — a Paper-API plugin using `io.papermc.paper.*` classes will throw `NoClassDefFoundError` on Spigot.
- **Folia (the inversion):** Folia REFUSES to load plugins unless they declare `folia-supported: true` in `plugin.yml` or `paper-plugin.yml`. This is the only Bukkit-family server that actively blocks plugins by default. The opt-in exists because Folia's regionized multithreading breaks the single-threaded assumptions of most existing plugins.

**Lucy's model:** Lucy's topology capability ladder mirrors this exactly. Each Bukkit-family node declares its highest rung; `RuntimeCapability.Populate()` (in `types/type_server_topology.go`) expands any declared rung into its full ancestry set. The registry stores only the highest rung; the materialized topology exposes the full superset so consumers asking "can this server host Bukkit plugins?" continue to work via `HasCapability(CapabilityBukkitPlugins)`.

Folia's compatibility inversion is enforced at the install/compat layer (a policy check), NOT at the capability-population layer (which stays descriptive). Folia populates through Paper (not Purpur) because Folia is a Paper fork.

**Excluded from having their own capability (have APIs but no plugin ecosystem targeting them):** Leaves (`org.leavesmc.leaves.*` — BotManager API, niche), Leaf (`org.dreeam.leaf.*` — 2 events), Pufferfish (deliberately no plugin-facing API), Airplane (discontinued).

**Sources:**
- Paper drop-in compatibility: https://docs.papermc.io/paper/getting-started/
- Purpur API inheritance: https://purpurmc.org
- Folia opt-in requirement: https://github.com/PaperMC/Folia README + `paper-patches/features/0003-Require-plugins-to-be-explicitly-marked-as-Folia-sup.patch`
- Folia scheduler docs: https://docs.papermc.io/paper/dev/folia-support/

### 6b. Hybrid Server Plugin-Tier Depth Varies

Hybrid servers (Forge/NeoForge/Fabric core + Bukkit plugin layer) do NOT all implement the same Bukkit rung. This matters for install compatibility — a Purpur-API plugin will run on Youer but NOT on Arclight.

| Hybrid | Mod loader | Bukkit tier implemented | Highest Bukkit capability |
|---|---|---|---|
| Arclight | Forge / NeoForge / Fabric (separate modules) | Bukkit + Spigot (Paper support in progress as of 2025) | `CapabilitySpigotPlugins` |
| CatServer | Forge | Bukkit + Spigot | `CapabilitySpigotPlugins` |
| Mohist | Forge (+ NeoForge) | Bukkit + Spigot | `CapabilitySpigotPlugins` |
| Youer | NeoForge | Bukkit + Spigot + Paper + Purpur (full chain) | `CapabilityPurpurPlugins` |

**Sources:**
- Arclight FAQ: https://github.com/IzzelAliz/Arclight/wiki/FAQ ("full support to Bukkit and Spigot plugins. Paper not yet.")
- Youer docs: https://mohistmc.cn/docs/youer (Purpur/Paper compatible)
- CatServer README, Mohist README

**Meta-cognition:** "This is a hybrid server" is not enough information to determine plugin compatibility. The Bukkit tier matters. Two hybrids with the same mod loader can have different plugin ecosystems.

### 7. Server Cores Are Not Mod Loaders, and Hybrids Are Neither

Server cores (Vanilla, Fabric server, Forge server, Paper, Purpur) are the runtime itself. They are distributed differently (direct JAR vs installer that emits multiple files) and don't follow mod/plugin metadata conventions. **Hybrid servers** (Arclight, Mohist, CatServer, Silkard, Youer) run a Forge/NeoForge/Fabric core and expose a Bukkit/Paper plugin API on top — they advertise identity via system properties, brand strings, and `@Mod` annotations that differ per fork.

Bridges like **Sinytra Connector** are NeoForge mods that load Fabric mods; they are not server cores and not mod loaders, but a third category. Geyser is distributed as seven different artifact forms (Bukkit plugin, BungeeCord plugin, Velocity plugin, Fabric mod, NeoForge mod, standalone, viaproxy) — each with a different metadata file and different ID rules.

**Lucy's first-class Core type:** Lucy distinguishes bootable server cores from non-core runtime layers via a `Core` enum (`types/type_core.go`). `Core` is an injective subset of `RuntimeNodeID` — every Core maps to exactly one node, but not every node is a Core:

- **Cores:** vanilla, mod loaders, plugin cores, hybrids, proxies, MCDR (21 values). Proxies are cores because they boot as standalone JVMs and future multi-server modelling requires every addressable server-process artifact to be installable.
- **Not cores:** bridges (Sinytra Connector, Kilt) and plugin-form protocol translators (Geyser-as-mod, Geyser-as-plugin). These run inside another core and cannot bootstrap a server on their own.

The `uint8` representation is intentional: it keeps `Core` type-distinct from the string-backed `RuntimeNodeID` so the compiler enforces the subset relationship at every use site. `CoreToNodeId` is the single chokepoint for the injective map.

See `docs/adr/2026-06-23-core-as-injective-subset-of-node-id.md` for the full rationale.

**Verify at:** `lucy/workspace/detector/` for the per-platform detector implementations. `lucy/types/type_core.go` for the Core enum and injective map. Identity signals are fork-specific and change across versions.

**Meta-cognition:** Don't conflate "what runs the server" with "what loads mods" with "what loads plugins". They are three orthogonal axes; hybrids occupy more than one. And don't conflate "is a runtime node" with "is installable as a primary server executable" — the former is structural, the latter is what `Core` captures.

### 8. Registries Have Different Capabilities

Don't assume every registry exposes search, dependency info, hash lookup, or platform metadata. Treat capabilities as discovered, not assumed.

| Capability | Modrinth | CurseForge | Hangar | MCDR Catalog | Spiget |
|---|---|---|---|---|---|
| Search | ✅ | ✅ | ✅ | ✅ (via GitHub raw) | ✅ |
| Fetch by version | ✅ | ✅ | ✅ | ✅ | ⚠️ exact-match only |
| Dependency listing | ✅ | ✅ | ✅ | ✅ (from local metadata) | ❌ |
| Hash-based artifact lookup | ✅ SHA-1 / SHA-512 | ⚠️ MurmurHash2 (custom, seed=1, whitespace-normalized) — NOT sha1 | ❌ | ❌ | ❌ |
| Platform support info | dynamic | ⚠️ incomplete (loader is mixed into `gameVersions[]`) | ✅ | ✅ | ✅ |
| Auth required for reads | ❌ | ✅ (`x-api-key`) | ❌ | ❌ | ❌ |

**Verify at:** `lucy/upstream/providers/{modrinth,curseforge,hangar,spiget,mcdr}/` — each provider declares its `Authentic` capability. The CurseForge fingerprint quirk (murmur2 with whitespace normalization) is implemented in `lucy/internal/artifacthash/file.go`.

**Meta-cognition:** "This registry has dependency info" is a property of the registry, not of the mod. Missing deps ≠ no deps.

## Common Mistakes

| Mistake | Fix                                                                                                                                                   |
|---|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Assuming mod ID equals registry slug | Treat them as separate namespaces. See [`package-names.md`](./package-names.md).                                                                      |
| Assuming one mod per JAR | Handle one-to-many artifact→identity mapping (Forge, NeoForge, Sponge).                                                                               |
| Assuming `id` is the identity field everywhere | Bukkit/BungeeCord use `name`. Forge ≤1.12 uses lowercase `modid`. Check per platform.                                                                 |
| Assuming versions are semver | Minecraft has its own scheme. Forge uses Maven ranges. MCDR uses a third custom syntax.                                                               |
| Assuming one version-range parser suffices | Dispatch by platform; see `lucy/version/version_range_dialect.go`.                                                                                    |
| Assuming all registries have dependency info | Spiget and MCDR Catalog differ. Capability must be checked per source.                                                                                |
| Assuming the CurseForge hash algorithm is sha1 | File metadata uses sha1/md5; the fingerprint match endpoint uses a custom murmur2. They are not interchangeable.                                      |
| Hardcoding registry URLs | Centralize URL construction. Endpoints and API versions change.                                                                                       |
| Assuming platform can be determined from one signal | Bukkit-family detection is multi-signal and probabilistic.                                                                                            |
| Treating server cores, hybrids, and bridges as the same kind of package | They install differently and define vs. run on the platform. Hybrids (Arclight/Mohist/Youer) and bridges (Sinytra Connector) are distinct categories. |
| Trusting the MCDR JSON schema over the runtime parser | Schema permits hyphens; Python runtime does not. Runtime wins.                                                                                        |
| Assuming `bungee.yml` is the BungeeCord descriptor filename | BungeeCord actually reads `plugin.yml`. The community calls it `bungee.yml` but the loader looks for `plugin.yml`.                                    |
| Treating all Bukkit-family servers as having the same plugin ecosystem | Bukkit → Spigot → Paper → Purpur is a strict inheritance ladder. Higher rungs host lower-rung plugins; the reverse fails at classload time. See §6a.   |
| Assuming Folia is just another Paper fork | Folia inverts compatibility — Paper plugins fail without `folia-supported: true`. The opt-in is enforced at install/compat, not at capability population. |
| Assuming all hybrids implement the same Bukkit tier | Arclight/CatServer/Mohist stop at Spigot; Youer implements the full Purpur chain. See §6b.                                                            |
| Assuming any RuntimeNode can be installed as a server core | Only nodes with a corresponding `Core` value (see `types/type_core.go`) are bootable. Bridges and plugin-form protocol translators are not cores.       |

## Lucy's Internal Model (Where to Look in This Repo)

When working inside lucy, these are the load-bearing types and where they live. Use this as an index, not a spec.

| Concept | Location | Notes                                                                                                                                                                                               |
|---|---|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Platform enum | `types/platform*.go` | `minecraft, fabric, forge, neoforge, mcdr, bukkit, sponge, velocity, bungeecord, none, unknown, any`. Subset declarable in manifest: `minecraft/fabric/forge/neoforge/mcdr/bukkit/any/none`.        |
| Source enum | `types/source*.go` | 12 values. CurseForge is conditional on cipher key being embedded at build time.                                                                                                                    |
| Package ref syntax | `input/syntax.go` | `scope:platform/name@version`. Identity aliases: `mc→minecraft`, `fabric-loader→fabric`, `mcdr→mcdreforged`. Only these five — nothing else is auto-normalized.                                     |
| Artifact reader priority | `artifact/reader.go` | 9 readers, first match wins. Order: fabric.mod.json → mods.toml → mcmod.info → neoforge.mods.toml → plugin.yml → velocity-plugin.json → bungee.yml → sponge_plugins.json → mcdreforged.plugin.json. |
| Version range dialects | `version/version_range_dialect.go` | NpmSemver (MCDR), FabricSemver (Fabric), MavenRange (Forge/NeoForge).                                                                                                                               |
| Core enum | `types/type_core.go` | First-class identity for bootable server cores. `uint8`, injective map to `RuntimeNodeID` via `CoreToNodeId`. 21 valid values + `CoreInvalid`. Bukkit-family Cores ordered by ancestry rank. See ADR `2026-06-23-core-as-injective-subset-of-node-id.md`. |
| Runtime topology | `types/topology*.go` + `workspace/detector/` | `RuntimeNode{ID, Role, Capabilities[], Identities[], RiskLevel}`. 7 roles, 5 risk levels (folds by max across edges).                                                                                  |
| Capability ladder | `types/type_server_topology.go` | 14 capabilities. Bukkit family is a ranked ladder: `CapabilityBukkitPlugins < CapabilitySpigotPlugins < CapabilityPaperPlugins < CapabilityPurpurPlugins`, with `CapabilityFoliaPlugins` branching off Paper (compat-inverting). `Populate()` expands any rung to its full ancestry. See ADR `2026-06-23-bukkit-capability-ladder-with-populate.md`. |
| Slugmap | `internal/slugmap/` | Remote-to-local slug overrides. Append-only. See [`package-names.md`](./package-names.md).                                                                                                          |
| CurseForge hash | `internal/artifacthash/file.go` | The murmur2 quirk. Don't replace with sha1.                                                                                                                                                         |
| Cipher / CF API key | `internal/cipher/` | ChaCha20Poly1305, build-time ldflags injection via `task cipher-generate` (needs `CF_API_KEY`).                                                                                                     |

## Revisit When

- A new major mod loader or plugin system gains significant adoption. (Recently: NeoForge split from Forge; Sinytra Connector emerged as a bridge category.)
- Mojang changes the server distribution model or metadata format.
- A cross-platform identity standard emerges in the modding community. (Currently: none. Don't hold your breath.)
- A registry adds or removes a capability (e.g., Hangar adds hash-based lookup, CurseForge changes the fingerprint algorithm). Update the capability matrix and the corresponding provider's `Authentic` flag.
- A hybrid server or bridge gains significant adoption. Add a detector and update §7.
- Folia's scheduler API is fully absorbed into Paper upstream. At that point `CapabilityFoliaPlugins` may become redundant and the inversion may disappear — the `Populate()` case for Folia would be removed. See ADR `2026-06-23-bukkit-capability-ladder-with-populate.md`.
- A Bukkit-family fork emerges with a plugin ecosystem significant enough to warrant its own capability constant (current bar: Purpur with 84+ Modrinth plugins and PurpurExtras at 28K downloads). Add the constant and a `Populate()` case delegating to its parent rung.
- Quilt (Fabric fork) gains adoption beyond niche use. Currently deferred — Quilt runs Fabric mods via QFAPI but Quilt-native mods don't run on Fabric. A capability split mirroring the Bukkit ladder may be warranted if Quilt-native plugins become significant.
