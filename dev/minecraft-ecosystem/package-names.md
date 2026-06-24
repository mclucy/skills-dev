# Local ID vs Remote Slug Mismatch

**This document is the deep dive for the cardinal sin:** the local ID a mod/plugin declares inside its artifact and the slug a registry (Modrinth, CurseForge, Hangar, Spiget) exposes to the world are **independently chosen by different parties at different times**. They are not aliases; they are separate namespaces that sometimes collide and often don't.

The main skill covers when to suspect this. This document covers *why* the divergence is structural, where to verify each side, and how lucy bridges them.

## Why IDs and Slugs Diverge — Structural Reasons

1. **Different authorities.** The local ID is chosen by the mod author and baked into the artifact. The slug is assigned (and reassignable) by the registry. CurseForge derives the slug from the project name at creation; Modrinth lets the author pick; Hangar uses `owner/slug`. None of them consult the local metadata.

2. **Slugs are renamable.** Both Modrinth and CurseForge allow project owners to rename a project, which changes the slug. The `project_id` (Modrinth) / numeric `id` (CurseForge) is immutable; the slug is not. A tool that cached `slug → mod` mapping will silently break on rename.

3. **Local IDs have different character rules.** A Fabric mod can use hyphens (`[a-z][a-z0-9-_]{1,63}`); a Forge mod cannot (`^[a-z][a-z0-9_]{1,63}$`, underscore only). CurseForge slugs allow hyphens. So a Fabric mod with id `my-cool-mod` cannot have its slug derived on Forge, and the CurseForge slug `my-cool-mod` cannot be a Forge modId directly.

4. **Bukkit doesn't have IDs at all.** Bukkit's `plugin.yml` uses `name` as the canonical identifier — and that name allows spaces and uppercase (`^[A-Za-z0-9 _.-]+$`). Bukkit also has no remote registry canonical to itself; Spiget and Hangar each apply their own slug rules.

5. **One artifact can declare multiple local IDs.** Forge / NeoForge / Sponge permit multiple `[[mods]]` / plugin entries per JAR. A registry lists one project per artifact. The local→remote mapping is many-to-one, never one-to-one in general.

## Where to Verify Each Side

Don't trust memory. Each side has an authoritative source.

### Local ID — by platform

| Platform | Authoritative file | Where in repo |
|---|---|---|
| Fabric | `fabric.mod.json` → `id` | [fabric-loader `MetadataVerifier.java`](https://github.com/FabricMC/fabric-loader/blob/trunk/src/main/java/net/fabricmc/loader/impl/metadata/MetadataVerifier.java) — regex `[a-z][a-z0-9-_]{1,63}` |
| Forge / NeoForge | `META-INF/mods.toml` → `modId` | [NeoForge `ModInfo.java`](https://github.com/neoforged/NeoForge/blob/26.2.x/fmlcore/src/main/java/net/neoforged/fml/core/ModInfo.java) — regex `^[a-z][a-z0-9_]{1,63}$` (underscore only) |
| Forge ≤1.12 | `mcmod.info` → `modid` (lowercase!) | [Forge 1.12 `ForgeBlockFlowParser`](https://github.com/MinecraftForge/MinecraftForge/blob/1.12.x/) — legacy, array-based |
| Bukkit / Spigot / Paper / Folia / Leaves | `plugin.yml` → `name` | [Bukkit `PluginDescriptionFile.java`](https://github.com/Bukkit/Bukkit/blob/master/src/main/java/org/bukkit/plugin/PluginDescriptionFile.java) — pattern `^[A-Za-z0-9 _.-]+$`, spaces→`_` |
| BungeeCord / Waterfall | `bungee.yml` (filename is actually `plugin.yml`) → `name` | [BungeeCord `ModuleManager.java`](https://github.com/SpigotMC/BungeeCord/blob/master/) |
| Velocity | `velocity-plugin.json` → `id` | [Velocity `JavaPluginLoader.java`](https://github.com/PaperMC/Velocity/blob/dev/3.0/proxy/src/main/java/com/velocitypowered/proxy/plugin/) — regex `[a-z][a-z0-9-_]{0,63}` |
| Sponge | `META-INF/sponge_plugins.json` → `plugins[].id` | [Sponge `PluginPlatformConstants.java`](https://github.com/SpongePowered/SpongeAPI/blob/main/src/main/java/org/spongepowered/api/plugin/) — kebab-case |
| MCDR | `mcdreforged.plugin.json` → `id` | [MCDReforged `metadata.py`](https://github.com/MCDReforged/MCDReforged/blob/master/mcdreforged/plugin/meta/metadata.py) — runtime regex `[a-z][a-z0-9_]{0,63}` (schema permits hyphens, runtime parser does NOT — schema is stale) |

### Remote slug — by registry

| Registry | Identity model | Slug rules | Renamable? | API |
|---|---|---|---|---|
| **Modrinth** | `project_id` (immutable base62) + `slug` (display) + `version_id` (per release) | `[a-zA-Z0-9._-]{3,64}` | **Yes** | [`api.modrinth.com/v2`](https://docs.modrinth.com/api/) |
| **CurseForge** | numeric `id` (immutable) + `slug` (derived from name) | lowercase, hyphens | **Yes** | `api.curseforge.com/v1` (no direct by-slug lookup — must search) |
| **Hangar** | `owner/slug` pair | project-scoped | **Yes** | `hangar.papermc.io/api/v1` |
| **Spiget** | numeric resource ID OR name (via search) | scraper of spigotmc.org | **No** (numeric ID is canonical) | `api.spiget.org/v2` |
| **MCDR PluginCatalogue** | `plugin_id` (from local metadata) | matches local MCDR id | **No** (id is the catalog key) | GitHub raw `MCDReforged/PluginCatalogue/contents/` |

## How Lucy Bridges Them

Lucy's resolver does not assume `local_id == slug`. The bridging is layered:

1. **Identity aliases are explicit and limited.** Only five bootstrap identities are normalized: `mc ↔ minecraft`, `fabric-loader ↔ fabric`, `mcdr ↔ mcdreforged`. Source: `lucy/input/syntax.go` and `lucy/internal/type_identity/type_identity.go`. Everything else is passed through verbatim.

2. **Hash-based lookup is the source of truth.** Modrinth `GET /v2/version_file/{sha1}` and CurseForge `POST /v1/fingerprints` (custom MurmurHash2, NOT sha1) bridge artifact→registry without consulting the local ID. Source: `lucy/upstream/providers/modrinth/` and `lucy/internal/artifacthash/file.go`.

3. **Slugmap for slug drift.** `lucy/internal/slugmap/` holds remote-to-local slug overrides when name-based matching is the only option (e.g., Spiget, MCDR PluginCatalogue). The slugmap is the only place where lucy explicitly says "this registry slug corresponds to this local ID" — and it is treated as fragile, append-only data.

4. **Per-server uniqueness is enforced at install time**, not at identity time. Lucy will refuse to install two packages with colliding local IDs into the same server directory, but it will happily install `mod-A` from Modrinth and `mod-A` from CurseForge into different servers even if they are different mods that happen to share a local ID.

## Signals Your Mapping Assumption Is Wrong

Stop and re-verify when you observe any of these:

- A search by slug returns zero results on a registry that should have the package → the slug was renamed.
- Two artifacts from different registries have the same slug but different hashes → they are different mods with colliding slugs.
- An artifact's parsed local ID differs from the registry slug → normal, do not "fix" by renaming either side.
- A Forge mod's `modId` contains a hyphen → invalid; the metadata is malformed or you parsed the wrong field (probably `namespace`, which does allow hyphens).
- A Bukkit plugin's `name` differs from any registry slug by case or spaces → expected; Bukkit `name` allows spaces and uppercase, registry slugs do not.
- A CurseForge project's `slug` differs from its website URL → CurseForge normalizes for URL display (lowercase, hyphens); the API `slug` is the source of truth.
- A Modrinth project URL doesn't match the `project_id` → URLs use slugs (renamable), `project_id` is the immutable key. Cache by `project_id`.
- MCDR catalog `plugin_id` disagrees with the schema regex → the runtime Python parser is stricter than the JSON schema. Trust the runtime parser.

## Meta-Cognition: The Default Posture

The default posture is **distrust, then verify**. Three concrete habits:

1. **When you read a local ID, do not assume the registry slug.** Look it up via hash or via the slugmap; do not derive it.
2. **When you cache a registry result, key it by the immutable identity** (`project_id` on Modrinth, numeric `id` on CurseForge, `owner/slug` on Hangar is acceptable only if owner is the canonical account). Slugs move.
3. **When a mapping breaks, treat it as new information about the world, not a bug in the data.** Add an entry to the slugmap; do not try to "repair" the registry or the local metadata.

The slugmap is append-only by design. Entries represent hard-won knowledge about specific divergences; they should accumulate, not be cleaned up.
