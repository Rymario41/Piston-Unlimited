# Piston-Unlimited: Minecraft 1.21.1 -> 26.3 / NeoForge

Analysis date: 2026-09-25. Branch inspected: `port/26.3`.

This is a planning deliverable. Implementation has NOT STARTED. No Java, build configuration, resources, dependencies, workflows, wrapper files, or gameplay have been changed. No compilation fixes, stubs, compatibility shims, or feature removals were attempted.

## 1. Scope, evidence, and estimate

The local codebase is small: **8 Java files, 1,137 lines including comments**, one block/item, one block entity, one GUI, and one serverbound configuration message. The port is nevertheless **medium in local implementation scope and high in integration/regression risk**. Nearly every class touches an API or library that changed across the requested version range. Porting the external libraries themselves would be separate, substantially larger work.

The inspection covered all tracked project files, both workflows, Java sources, resource contents, wrapper scripts/configuration, the empty access transformer, existing build artifacts, cached dependency JARs/POMs/Gradle metadata, and the remotely applied build script. Generated/decompiled Minecraft files, caches, and runtime directories are build evidence, not additional project source to port.

**Baseline-build checkpoint: COMPLETED.** The user has already manually verified the Minecraft 1.21.1 build with `./gradlew.bat build` under **JDK 21**, obtaining **BUILD SUCCESSFUL**. This checkpoint is accepted as completed and is not unfinished work or a prerequisite to repeat in Stage 0. Existing `build/libs/multipiston-0.0.1-1.21.1.jar` and compiled classes are also present; the inspected class has Java class-file major version 65 (Java 21). No additional Gradle task or game launch was run during this documentation-only work.

Official sources were checked live. Target library availability and exact patched signatures still need validation against a resolved 26.3 workspace. A primer describes an intermediate transition; it does not guarantee that the intermediate API survives through 26.3.

## 2. Current architecture and complete project inventory

Single Gradle project, `rootProject.name = "multipiston"`; package/group `com.ldtteam`; mod ID `multipiston`; display name `Multi-Piston`. There are no subprojects or separate platform modules.

For the tables/stages below, `J` means `src/main/java/com/ldtteam/multipiston`, and `R` means `src/main/resources`. These are repository-relative paths.

| File | Current responsibility and migration surface |
| --- | --- |
| `J/MultiPiston.java` | `@Mod` entry point; receives `FMLModContainer` and `Dist`; attaches deferred block/item/block-entity/tab registers; static payload registration through `RegisterPayloadHandlersEvent`; network version is the built mod version. |
| `J/ModBlocks.java` | Registers `multipiston:multipistonblock` as both block and block item. Supplier-based block construction and `new Item.Properties()` currently have no explicitly assigned registry keys. |
| `J/ModTileEntities.java` | Registers `multipiston:multipistonte` through `BlockEntityType.Builder.of(...).build(null)`. |
| `J/MultiPistonBlock.java` | Stone-like `BaseEntityBlock`, collision/selection shapes, GUI interaction with/without a held item, redstone neighbor callback, block entity factory/ticker, model render shape. `codec()` currently returns null. |
| `J/TileEntityMultiPiston.java` | Server tick and redstone state machine; moves rows of matching blocks between chosen directions; moves obstructing entities; copies allowed block entities; persistence and client updates; Structurize rotation/mirroring. Largest class: 437 lines. |
| `J/AbstractWindowSkeleton.java` | BlockUI `BOWindow`, resource identifier construction, button dispatch and logging. |
| `J/WindowMultiPiston.java` | BlockUI XML controls, six direction/color choices, range/speed input, error sound through Structurize, optimistic client update and send-to-server action. |
| `J/network/MultiPistonChangeMessage.java` | BlockUI-provided `com.ldtteam.common.network` abstraction. Serializes block position, input/output direction ordinals, range, speed; changes server block entity and calls `sendBlockUpdated(..., 0x3)`. |

The `com.ldtteam.common.network.AbstractServerPlayMessage` and `PlayMessageType` classes were located inside the cached **BlockUI JAR**. They are not locally missing classes and must not be replaced with fake implementations.

Resources:

- `R/META-INF/neoforge.mods.toml`: loader metadata and four required dependencies (`neoforge`, `minecraft`, `blockui`, `structurize`), all on BOTH sides; library ordering AFTER. Expanded by the shared Gradle script.
- `R/META-INF/accesstransformer.cfg`: present, **zero bytes**. No active access-widening rules to translate.
- `R/assets/multipiston/blockstates/multipistonblock.json`: one default model variant.
- `R/assets/multipiston/models/block/multipistonblock.json`: 64-element Blockbench model, vanilla andesite/concrete textures, some existing `#missing` face references. Preserve geometry, colors and proportions; distinguish pre-existing texture warnings from new failures.
- `R/assets/multipiston/models/item/multipistonblock.json`: block-model parent plus hand, inventory, ground, head and fixed transforms. No modern `assets/multipiston/items/multipistonblock.json` definition exists yet.
- `R/assets/multipiston/gui/windowmultipiston.xml` and `R/assets/multipiston/textures/gui/builder_button_medium.png`: one BlockUI window and its existing button texture.
- `R/assets/multipiston/lang/en_us.json`: block name and equal-direction error translation.
- `R/data/multipiston/recipe/multiblock.json`: shaped recipe, 3 stone + 2 redstone blocks + 3 pistons -> 1 multipiston; legacy ingredient objects and an obsolete `data: 0` field on stone.
- `R/data/multipiston/loot_table/blocks/multipistonblock.json`: self-drop with explosion condition.
- `R/data/minecraft/tags/block/mineable/pickaxe.json`: pickaxe tag membership.

No legacy `mods.toml`, mixin configuration/classes, coremods, custom renderers/shaders, custom recipe serializers, custom entities, world generation, commands, capabilities, config specification, menus, or local datagen providers were found. No tracked tests/GameTests, generated resources, `pack.mcmeta`, dependency locks, or IDE launch files were found. Do not invent migrations for systems this mod does not implement. Renderer changes primarily affect its dependencies and JSON resources.

Other files: `README.md` is only the project name; `LICENSE.txt` and `CLA.md` do not need porting. `.gitignore` excludes build/cache/run/IDE directories and `gradle/local.gradle`. The two GitHub workflows delegate build/prerelease/publish to OperaPublicaCreator `@ng7` and select Java 21. Publishing is conditional; this plan does not authorize publishing.

### Behavior that must survive the port

- Registry IDs, mod ID, translations, crafting inputs/output, pickaxe classification, visual appearance, GUI controls, and rotation/mirror integration.
- Default range 3, default speed 2, upper range 10, speed clamp 1..3, and the existing integer tick interval `20 / speed`.
- Server-side movement; redstone switches direction only when the previous movement has completed. Existing handling of incomplete motion on load must be compared with the source, not silently redesigned.
- Filtering of air, bedrock and IGNORE/DESTROY/BLOCK push reactions; ordinary block entities are excluded, with the existing `domum_ornamentum` namespace exception retained.
- Matching block type checks, loaded-chunk checks, liquid destinations, neighbor-shape updates, source removal order, entity displacement, sounds, and update flag semantics.
- Existing saved keys and types: `input` boolean, `range`, `direction`, `progress`, `outputDirection`, `speed` integers. `length` is declared but not written. Missing `outputDirection` falls back to the opposite input. Directions currently use ordinals, not names.

`currentDirection`, `ticksPassed` and the timing of dirty marking are baseline behaviors to observe. Current setters/network handling do not explicitly call `setChanged()`. Malformed direction ordinals and range lower bounds also deserve baseline comparison, but this is not authorization for unrelated gameplay or security refactors. If a defect blocks correct persistence on the new version, report it separately and make the smallest justified change.

## 3. Versions, Java, Gradle, and hidden build responsibilities

| Setting | Source project | Target reference / decision |
| --- | --- | --- |
| Minecraft | `exactMinecraftVersion=1.21.1`, `minecraftVersion=1.21.1`; additional publishing version `1.21` | `26.3`; do not keep advertising 1.21 compatibility |
| NeoForge | `forgeVersion=21.1.20` (property name is historical; loader is NeoForge) | MDK reference `26.3.0.10-beta`; recheck and pin a tested 26.3 build before implementation |
| Declared MC range | `[1.21, 1.22)` | MDK uses `[26.3]` |
| Declared NeoForge range | `[21.0.143,)` | Use the verified minimum target build; do not leave the source minimum |
| FML range | `[4,)`, used by `loaderVersion` | Follow 26.3 metadata schema; its MDK omits `modLoader` and `loaderVersion` |
| Mod version | base `0.0.1`, shared script produces `0.0.1-1.21.1` | Preserve deliberate version/publishing semantics with a 26.3 suffix; do not copy `examplemod` defaults |
| Wrapper | **Gradle 8.13**, BIN distribution | **Gradle 9.2.1**, as inspected in the 26.3 NeoGradle MDK |
| Gradle plugin | remotely selected NeoGradle userdev **`7.0.+`**, not pinned locally | Recommended **`net.neoforged.gradle.userdev` 7.1.39**, MDK reference |
| Source Java | **21**, toolchain enabled; CI Java 21; existing class major 65 | **25** for compilation, runtime, IDE and CI |
| Shell Java observed | Oracle Java **25.0.4** | Suitable language generation for target; this is distinct from source toolchain/Gradle daemon |
| Toolchain resolver | Foojay convention `0.5.0` | MDK `1.0.0` |
| Mappings | Parchment MC `1.21`, version `2024.07.28` | Official unobfuscated 26.3 names; no 1.21 Parchment overlay |

The source build has already passed under JDK 21; no baseline rerun is pending. The Java 25 executable observed in the shell does not change that verified source environment. Java toolchain selection and the JVM running Gradle are separate concerns. NeoForge's [26.1 release guidance](https://neoforged.net/news/26.1release/) establishes Java 25, Gradle >=9.1 and removal of the need for Parchment after deobfuscation; the actual 26.3 MDK supplies the target versions above.

### Remote build script

The current `build.gradle` **delegates its build logic to the remote LDTTeam OperaPublicaCreator `mod.gradle` script**, through `apply from: 'https://raw.githubusercontent.com/ldtteam/OperaPublicaCreator/ng7/gradle/mod.gradle'`. The [remote ng7 script](https://raw.githubusercontent.com/ldtteam/OperaPublicaCreator/ng7/gradle/mod.gradle) is on a mutable branch. The source project's actual build is much larger than that one line:

- It sets toolchains, NeoForge userdev, source sets, archive/version behavior, AT handling, resource expansion, JAR/source/Javadoc tasks, and publication facilities.
- It reads `gradle/*.gradle`, including `gradle/dependencies.gradle`, and optional local overrides; no local override was found here.
- It supplies Maven Central, Maven Local, LDTTeam JFrog, and `libs` resolution, plus plugin repositories.
- It defines `client`, `server`, `data`, `gameTestServer`; datagen writes `src/datagen/generated/multipiston`, also included as resources. There is no local generator implementation.
- It has legacy test and publishing plugin dependencies. A modern build must account for these responsibilities instead of only changing `forgeVersion`.

**Migration risk:** compatibility of this delegated build system with Minecraft 26.3, modern NeoGradle, Gradle 9 and Java 25 must be established. If replacement is necessary, its used dependency, run, resource-processing, packaging and publication responsibilities must be preserved explicitly. A successful 1.21.1 build does not establish target build-system compatibility. Stage 0 confirms the migration route; Stage 1 implements the selected build-system adaptation/replacement.

These are observations of the remote branch on the analysis date, not a guaranteed immutable reconstruction of the manually verified source build. Record a fixed reference when implementing; the completed baseline-build checkpoint remains valid.

## 4. Official 26.3 MDK comparison

Recommended base: retain **NeoGradle**, using an explicit local build modeled on the official [26.3 NeoGradle MDK](https://github.com/NeoForgeMDKs/MDK-26.3-NeoGradle/tree/4cb9b266eb4f5b9c2565a16022a61bd42c412c79). Inspected revision: `4cb9b266eb4f5b9c2565a16022a61bd42c412c79`. This avoids changing both Minecraft and the Gradle plugin family simultaneously.

Reference files: [build.gradle](https://github.com/NeoForgeMDKs/MDK-26.3-NeoGradle/blob/4cb9b266eb4f5b9c2565a16022a61bd42c412c79/build.gradle), [settings.gradle](https://github.com/NeoForgeMDKs/MDK-26.3-NeoGradle/blob/4cb9b266eb4f5b9c2565a16022a61bd42c412c79/settings.gradle), [gradle.properties](https://github.com/NeoForgeMDKs/MDK-26.3-NeoGradle/blob/4cb9b266eb4f5b9c2565a16022a61bd42c412c79/gradle.properties), [wrapper](https://github.com/NeoForgeMDKs/MDK-26.3-NeoGradle/blob/4cb9b266eb4f5b9c2565a16022a61bd42c412c79/gradle/wrapper/gradle-wrapper.properties), [metadata](https://github.com/NeoForgeMDKs/MDK-26.3-NeoGradle/blob/4cb9b266eb4f5b9c2565a16022a61bd42c412c79/src/main/resources/META-INF/neoforge.mods.toml).

Its structure uses explicit plugins, `implementation "net.neoforged:neoforge:${neo_version}"`, keyed metadata expansion, `src/generated/resources`, and per-run source binding. Runs are `client`, `server` (nogui), `gameTestServer`, and **`clientData`**. GameTest namespace property is `neoforge.enabledGameTestNamespaces`. Keep this project's identity and required libraries; do not copy sample content or the example publishing destination. The existing resource metadata location remains valid for NeoGradle.

The official [26.3 ModDevGradle MDK](https://github.com/NeoForgeMDKs/MDK-26.3-ModDevGradle/tree/7be5660e6c9131a82725aa6f1e4922c0f40475ee) was also inspected, at revision `7be5660e6c9131a82725aa6f1e4922c0f40475ee`: plugin `net.neoforged.moddev` **2.0.147**, `neoForge { version; runs; mods }`, and `src/main/templates` metadata expanded by `generateModMetadata`. Its run named `data` invokes `clientData()`. This is an alternative, not a second plugin to apply alongside userdev. Moving metadata to templates is therefore **not mandatory** for the recommended NeoGradle route.

Neither inspected target layout requires adding an arbitrary hand-written `pack.mcmeta`. Verify loader-provided pack metadata before introducing one; do not guess pack format numbers.

## 5. Official migration chain and applicability

The following sequence covers every requested release boundary. Read the vanilla primer and its linked Neo Changes article where available. Apply the cumulative changes directly to 26.3 in small subsystem stages; intermediate full release builds are optional troubleshooting checkpoints, not prerequisites. There is no separate 1.21.3 primer: 1.21.2/3 is one upstream transition.

| Transition / official reference | Relevance to this repository |
| --- | --- |
| [1.21.1 -> 1.21.2/3](https://docs.neoforged.net/primer/docs/1.21.2/) / [NeoForge](https://neoforged.net/news/21.2release/) | Block/item properties require registry IDs; use deferred registration helpers that supply them. `BlockEntityType.Builder` disappears. `ItemInteractionResult` merges into `InteractionResult`. Neighbor updates introduce `Orientation`. Ingredient/recipe representations and GUI rendering APIs change. Direct impact: registries, block interactions, movement callbacks; indirect: BlockUI. |
| [1.21.2/3 -> 1.21.4](https://docs.neoforged.net/primer/docs/1.21.4/) / [NeoForge](https://neoforged.net/news/21.4release/) | Item model definitions move behind `assets/<namespace>/items/<id>.json`; the old model remains a referenced geometry model. Datagen distinguishes client/server entry points and `GatherDataEvent` variants. Preserve transforms and select the proper target data run. |
| [1.21.4 -> 1.21.5](https://docs.neoforged.net/primer/docs/1.21.5/) / [NeoForge](https://neoforged.net/news/21.5release/) | CompoundTag getters become Optional/default-based and key access changes; block entity removal lifecycle and rendering/model APIs change. Direct risk: state copy during movement and default values; indirect risk: GUI and Structurize renderers. |
| [1.21.5 -> 1.21.6](https://docs.neoforged.net/primer/docs/1.21.6/) / [NeoForge](https://neoforged.net/news/21.6release/) | Higher-level persistence moves to `ValueInput`/`ValueOutput`, with tag adapters and problem reporting. Migrate block entity save/load, update-tag bridges and moved-block data together. GUI extraction/render-state changes affect BlockUI. |
| [1.21.6 -> 1.21.7](https://docs.neoforged.net/primer/docs/1.21.7/) | GUI highlighting and item-render-state adjustments; no matching direct renderer calls in this mod. Check dependency GUI behavior; do not invent a piston algorithm rewrite for this boundary. |
| [1.21.7 -> 1.21.8](https://docs.neoforged.net/primer/docs/1.21.8/) | Primer lists graphics workarounds; no identified direct source migration. Still included in dependency/runtime verification. |
| [1.21.8 -> 1.21.9](https://docs.neoforged.net/primer/docs/1.21.9/) / [NeoForge](https://neoforged.net/news/21.9release/) | `Level.isClientSide` becomes private: use the accessor. GUI key/mouse input becomes event objects. Rendering submission and typed block-entity item data change. Direct: side guards; indirect: BlockUI input and dependency rendering/state transfer. |
| [1.21.9 -> 1.21.10](https://docs.neoforged.net/primer/docs/1.21.10/) | `entityInside` gains a boolean and hanging-entity collision behavior changes. No local override to migrate. Verify displaced entities behavior instead of modifying nonexistent hooks. |
| [1.21.10 -> 1.21.11](https://docs.neoforged.net/primer/docs/1.21.11/) / [NeoForge](https://neoforged.net/news/21.11release/) | `ResourceLocation` becomes `Identifier`; package moves, nullability annotations and renderer/atlas changes. Direct: window resource identifier; indirect: libraries, model texture resolution. Only change imports actually used. |
| [1.21.11 -> 26.1](https://docs.neoforged.net/primer/docs/26.1/) / [NeoForge](https://neoforged.net/news/26.1release/) | Java 25/deobfuscation, new NeoForge version numbering, recipe stack templates/common info, changed model/material machinery and GUI extraction names. Build/toolchain and dependencies are major work; no custom Java recipe/renderer exists here to port. |
| [26.1.x -> 26.2](https://docs.neoforged.net/primer/docs/26.2/) | Vulkan/backend and GUI reorganization, registry ID classes and vanilla registry-object separation. Primarily dependency compatibility; audit any referenced registry constants against 26.3 rather than globally rewriting every registration. |
| [26.2 -> 26.3](https://docs.neoforged.net/primer/docs/26.3/) | SDL input and Renderpearl client changes affect BlockUI. Recipes/loot become reloadable datapack registries. Block type codec registry and `Block#codec` are removed: the existing null-returning override must eventually be removed as an obsolete API, not replaced with a dummy codec. Resource loading/reloading needs explicit validation. |

The upstream [primer index](https://docs.neoforged.net/primer/docs/) is the authority for the chain. Its primers are non-exhaustive; target NeoForge-patched source is the final authority for overloads and constructor signatures.

## 6. Major API areas and expected end state

| Area | Current API/behavior | Expected 26.3 treatment and principal risk |
| --- | --- | --- |
| Loader / lifecycle / events | Concrete `FMLModContainer`, event bus from container, static `@SubscribeEvent` method | Align entry point with MDK's injected `IEventBus` / `ModContainer`; confirm supported event subscription and exactly-once payload registration. Avoid client class loading on dedicated server. |
| Registry construction | No-arg custom block supplier; plain item properties; block entity builder | Properties supplied with IDs via target deferred helpers; factory accepts properties. Use actual NeoForge-accessible block entity type construction, retaining supported blocks and IDs. Bootstrap can fail before the game opens. |
| Block API and interaction | Two result types; old neighbor-position argument; direct side field; null codec | Unified results, target neighbor context/signatures and side accessor. Delete obsolete codec override only when implementing 26.3. Preserve both GUI opening paths and interaction consumption. |
| Persistence and synchronization | CompoundTag plus HolderLookup callbacks, direction ordinals, implicit defaults | Target ValueInput/ValueOutput and actual packet/update-tag bridges; explicit compatible defaults and registry-aware copying. Saved data must retain meaning and numeric direction mapping. |
| Block movement / redstone | `updateFromNeighbourShapes`, `setBlock(...,67)`, old `neighborChanged`, `BucketPickup`, remove/copy order | Match final overloads and named flag meanings; preserve source algorithm and checks. Highest risk of duplication, data loss, redstone recursion or fluid regressions. |
| Entity displacement / sound | AABB query and `teleportTo(x,y,z)`; piston extend sound | Verify the used overload and server behavior rather than assuming all teleport methods changed equally. Preserve destination and sound timing. |
| Networking | BlockUI message abstraction, registry-aware buffer, versioned NeoForge registrar | Use target BlockUI transport and actual payload APIs; preserve payload ID, field order and direction encoding unless a documented protocol change is necessary. Confirm execution on the server game thread and update delivery to observers. |
| GUI / identifiers / side isolation | BOWindow, Button/TextField/DropDownList, ResourceLocation, common block references client window | Real 26.3 BlockUI APIs, Identifier and proper client entry boundary if needed. Keep XML controls and rendering/input behavior. SDL/key changes belong principally in BlockUI. |
| Structurize integration | IRotatableBlockEntity, RotationMirror, Utils.playErrorSound | Use compatible upstream equivalents; preserve rotation then mirroring, vertical directions and error feedback. No local facsimile of the library. |
| Resource models / textures | Legacy inventory model discovery, static block model | Add item-definition indirection; verify 26.x material/atlas handling, face textures and all item transforms. No art redesign or placeholder texture. |
| Recipes / loot / tags | Hand-authored JSON and singular 1.21 directories | Convert obsolete ingredient encoding, validate target result/template and loot codecs against vanilla 26.3 examples. Preserve recipe/drop semantics. No Java recipe serializer or registry added for this simple recipe. |
| Datagen / runs / testing | Shared `data` run, absent local providers, no local tests | MDK-compatible client-data run, explicit output/source inclusion, modern GameTest namespace. An empty datagen/test run is not evidence that gameplay works. |
| Access transformation / mixins | Empty AT, no mixins | Nothing to remap; keep absence of active rules. Do not add broad access transformers or mixins to mask dependency incompatibility. |
| Build / CI / distribution | Mutable remote Gradle and CI scripts, old plugin suite, Java 21 | Explicit pinned modern configuration, Java 25 CI, preserved artifacts/dependencies and controlled metadata expansion. Validate without publishing. |

## 7. Dependencies and feasibility gate

| Dependency | Current evidence | Required action |
| --- | --- | --- |
| NeoForge | `net.neoforged:neoforge:21.1.20`, supplied through userdev configuration | Pin a tested 26.3 build; MDK baseline is `26.3.0.10-beta`. Beta APIs may still change. |
| BlockUI | `com.ldtteam:blockui:1.0.188-1.21.1-snapshot`; required for GUI **and network base classes** | Obtain an actual 26.3-compatible artifact and source/API reference; confirm BOTH-side loading. No verified 26.3 coordinate identified during this audit. |
| Structurize | `com.ldtteam:structurize:1.0.751-1.21.1-snapshot`; rotation/mirror and error sound | Obtain a compatible artifact matched to BlockUI. No verified 26.3 coordinate identified. |
| Domum Ornamentum | Cached Structurize runtime metadata: `com.ldtteam:domum-ornamentum:1.0.203-1.21.1-snapshot` | Verify target transitive resolution and moved-block data preservation. The namespace exception in the piston makes this a real integration test requirement. |
| LDTTeam datagenerators | Cached Structurize runtime metadata: `com.ldtteam:datagenerators:1.20.4-0.1.57-ALPHA`, universal classifier in POM | Determine the target Structurize dependency contract. Do not carry a 1.20.4 binary into 26.3 or exclude it merely to silence loader errors. |
| Parchment | `1.21 / 2024.07.28` | Retire the old mapping overlay with the deobfuscated target toolchain. This is tooling replacement, not gameplay removal. |
| JetBrains annotations | Cached runtime metadata uses `org.jetbrains:annotations:21.0.1` | Check whether target dependencies provide them or declare an appropriate explicit compile dependency; align required nullability signatures without unnecessary rewrites. |
| Guava / logging | Used through platform/dependency classpath | Verify actual target compile classpath; no need to shade or bundle replacements merely because imports exist. |
| Legacy build/test plugins | Shared script includes Shadow 7.0.0, download 4.1.2, Crowdin 0.5.0, Sonar 3.3, grgit 5.+ and CurseForgeGradle 1.1.25; some conditional | Audit Gradle 9 compatibility and preserve any active build/release responsibility. Do not blindly import the whole old plugin suite. |
| Legacy test suite | Shared script configures JUnit 4.13, Mockito 1.+, PowerMock 2.0.2, AssertJ 3.9.0, Hamcrest 1.3; no local tests | Select Java-25-compatible tooling for real future tests. Distinguish test variants from production runtime dependencies. |

Availability check, not a blanket claim of absence: upstream [BlockUI port/26 properties](https://github.com/ldtteam/BlockUI/blob/port/26/gradle.properties) currently target **26.1.2**, while [version/26](https://github.com/ldtteam/BlockUI/blob/version/26/gradle.properties) targets an earlier 26.1 snapshot. [Structurize version/1.21](https://github.com/ldtteam/Structurize/blob/version/1.21/gradle.properties) still targets **1.21.1**. The public [BlockUI Maven metadata endpoint](https://ldtteam.jfrog.io/artifactory/modding/com/ldtteam/blockui/maven-metadata.xml) checked did not establish a 26.3 release; the corresponding Structurize response did not provide usable Maven XML. Branch names, broad version ranges, or successful downloads do not establish binary compatibility.

**Gate:** identify compatible libraries and their transitive versions before committing to the implementation schedule. If unavailable, record the blocker and coordinate separate upstream ports. Do not remove GUI, networking, rotations, Domum support or dependencies, create stubs, or label a nonfunctional build as ported.

## 8. Small implementation stages

All commands below are **future verification commands**, not commands executed in this documentation-only task. They assume PowerShell from the repository root and the recommended NeoGradle route. Build/runtime stages require explicit implementation authorization after this plan. Do not publish artifacts as a verification step.

During stages 3-7, Java files are interdependent; `compileJava` can still report errors belonging to later stages. Record those diagnostics without patching unrelated areas. These stages are not all independently runnable game builds. Final compilation must be clean before runtime acceptance. No errors may be hidden through exclusions, disabled features, stubs or suppressed tasks.

### Stage 0 — Confirm the 26.3 migration path and dependency compatibility (FIRST)

- **Files affected:** PORTING_PLAN.md and PORTING_STATUS.md only for evidence; inspect all build/dependency declarations without changing source.
- **Old -> new:** completed, manually verified 1.21.1 baseline with unconfirmed target migration route/library set -> confirmed 26.3 migration route and a concrete compatible dependency matrix. Baseline build reproduction is outside this stage and is not pending.
- **Changes:** confirm the official migration chain and MDK-based build-system route, including compatibility or replacement of the remote OperaPublicaCreator script; identify exact BlockUI/Structurize/Domum/datagen builds, target APIs, artifact repositories and fixed source references. This stage records findings only and makes no implementation changes.
- **Risks:** target dependencies may not exist yet; Maven Local/caches can hide missing publications; one updated direct dependency may bring old transitive binaries; the delegated build system may require replacement to support the target toolchain.
- **Verification commands:** `Get-Content build.gradle` to confirm the remote build delegation; `jar tf <verified-target-jar>` for real candidate target artifacts, followed by read-only inspection of their metadata/classes and published POM/Gradle metadata. Replace the placeholder only after identifying an actual artifact. Compare official migration/MDK references and dependency API sources; no baseline-build command is required.
- **Exit:** document the selected migration/build-system route, actual dependency coordinates and compatibility evidence, or record the named target blockers. Do not infer compatibility from an allowed Minecraft range. The baseline-build checkpoint remains COMPLETED.

### Stage 1 — Establish the 26.3 build toolchain

- **Files affected:** `build.gradle`, `settings.gradle`, `gradle.properties`, `gradle/wrapper/gradle-wrapper.properties`, `gradle/wrapper/gradle-wrapper.jar`, `gradlew`, `gradlew.bat`; `gradle/dependencies.gradle` only as needed to keep its application explicit.
- **Old -> new:** remotely applied ng7 / dynamic userdev / Gradle 8.13 / Java 21 / Parchment -> pinned MDK-based NeoGradle / Gradle 9.2.1 / Java 25 / official names.
- **Changes:** replace the remote build apply with explicit configuration while retaining its used capabilities; select the target NeoForge version, settings repositories and Foojay resolver; regenerate the complete wrapper consistently. Keep mod identity and artifact naming. Do not copy MDK Java classes.
- **Risks:** the remote LDTTeam OperaPublicaCreator build system and its legacy plugins may be incompatible with the target toolchain; replacing it can lose hidden run/resource/packaging/publication behavior. Custom property mutation in settings may conflict with Gradle 9/configuration cache; changing JVM and wrapper in the wrong order can prevent startup.
- **Verification commands:** `.\gradlew.bat --version`; `.\gradlew.bat help --no-configuration-cache`; `.\gradlew.bat javaToolchains --no-configuration-cache`; `.\gradlew.bat tasks --all --no-configuration-cache`.
- **Exit:** target Gradle/configuration succeeds and selects Java 25. A working configuration is not a successful mod compilation.

### Stage 2 — Resolve libraries and modernize loader metadata

- **Files affected:** `gradle.properties`, `gradle/dependencies.gradle`, `build.gradle`, `R/META-INF/neoforge.mods.toml`. Keep the empty AT intact unless a concrete target requirement arises.
- **Old -> new:** 1.21.1 direct/transitive artifacts and broad old loader ranges -> verified 26.3 dependency graph and correctly expanded metadata.
- **Changes:** pin Stage 0's real versions, establish modern resource expansion including every current placeholder, align ranges/loader fields with MDK, keep BOTH-side requirements and ordering. Preserve author/license/mod identity. Record the existing missing `examplemod.png` reference as metadata cleanup if addressed, not as a new texture task.
- **Risks:** unresolved placeholders, duplicate library versions, accidentally client-only dependency declarations, old transitive Minecraft code, incorrect publishing compatibility labels.
- **Verification commands:** `.\gradlew.bat dependencies --configuration runtimeClasspath --no-configuration-cache`; `.\gradlew.bat dependencyInsight --dependency blockui --configuration runtimeClasspath --no-configuration-cache`; repeat for `structurize`, `domum-ornamentum` and `datagenerators`; `.\gradlew.bat processResources --no-configuration-cache`; `Get-Content build/resources/main/META-INF/neoforge.mods.toml`.
- **Exit:** no unresolved target artifacts or template tokens; inspect the whole runtime graph, not just direct declarations.

### Stage 3 — Port entry point, registries and the block API shell

- **Files affected:** `J/MultiPiston.java`, `J/ModBlocks.java`, `J/ModTileEntities.java`, `J/MultiPistonBlock.java`.
- **Old -> new:** legacy container construction, unkeyed properties, block entity builder and old block callbacks -> target injection, keyed factories and actual 26.3 signatures.
- **Changes:** use target event/registration APIs, retain IDs, pass block properties through the constructor, register block item through a key-aware helper, adapt block entity type creation. Migrate both interaction methods and the neighbor callback, keep tick/shape behavior, use side accessor. Remove the obsolete `codec()` override/import according to 26.3; do not replace its null with a fake compatibility implementation.
- **Risks:** bootstrap exceptions, wrong drop/translation IDs, duplicate event listeners, GUI opening twice or vanilla held-item behavior changing.
- **Verification command:** `.\gradlew.bat compileJava --no-configuration-cache`. Categorize remaining errors in persistence/network/UI for later stages.
- **Exit:** registration and block API mismatches addressed; actual placement/creative-tab checks wait for a complete compilable target.

### Stage 4 — Port persistence, sync tags and block-entity copying

- **Files affected:** `J/TileEntityMultiPiston.java`, especially load/save, update tag/packet handlers and the move-copy block.
- **Old -> new:** CompoundTag callbacks with registry provider -> ValueInput/ValueOutput and the target's supported tag/update bridges.
- **Changes:** retain keys, types, defaults and ordinal mapping; adapt registry-aware save/load copying to the destination entity; verify components, type identity, coordinates and update packet content. Use target adapters/problem reporting where actual signatures require them.
- **Risks:** lost direction/speed, wrong missing-field defaults, component loss, destination referencing source coordinates, stale client state, silent unsaved changes.
- **Verification command:** `.\gradlew.bat compileJava --no-configuration-cache`; once runtime-ready, `.\gradlew.bat runClient` and compare `/data get block <x> <y> <z>` before/after movement, save/reload and reconnection on copied fixtures.
- **Exit:** round-trip persisted state and copy/sync behavior match the source, including old records without `outputDirection`. No top-level save format or save-version redesign.

### Stage 5 — Port redstone, movement and entity interactions

- **Files affected:** `J/MultiPistonBlock.java`, `J/TileEntityMultiPiston.java`.
- **Old -> new:** old Level/neighbor/shape/teleport/fluid entry points -> final 26.3 equivalents with the same state-machine decisions.
- **Changes:** verify neighbor Orientation/context, update flags, shape update calls, chunk bounds, fluid pickup, sound and entity movement overloads. Preserve source matching, filtering and destination-first/copy/source-removal sequencing unless target mechanics require a documented equivalent.
- **Risks:** block duplication/loss, redstone feedback, moved block entity data loss, collision/teleport changes, fluid loss beyond baseline, chunk-boundary errors and speed regression.
- **Verification commands:** `.\gradlew.bat compileJava --no-configuration-cache`; after Stage 7, `.\gradlew.bat runClient` and `.\gradlew.bat runServer` with the movement matrix in section 9.
- **Exit:** no unaccounted behavior differences for all six directions, range/speed boundaries and supported/blocked material cases.

### Stage 6 — Port payload registration and server configuration updates

- **Files affected:** `J/network/MultiPistonChangeMessage.java`, network registration in `J/MultiPiston.java`, send call in `J/WindowMultiPiston.java` if the real library API requires it.
- **Old -> new:** source BlockUI message wrappers and registrar -> actual compatible target wrappers/payload contract.
- **Changes:** retain `multipiston:change_msg`, logical fields and encoding order; match target registration, buffer and handler signatures. Verify thread dispatch and client/server protocol version agreement; retain the existing meaning of update flag `0x3`.
- **Risks:** changed direction encoding, handshake rejection, decode errors, execution off the game thread, settings updating only the initiating client.
- **Verification commands:** `.\gradlew.bat compileJava --no-configuration-cache`; after Stage 7, `.\gradlew.bat runServer` plus `.\gradlew.bat runClient` and a second test client with matching libraries.
- **Exit:** changing all settings reaches the server and another observing client and survives reopen/reconnect as the source does. Do not substitute homemade library classes.

### Stage 7 — Port the BlockUI/Structurize client integration

- **Files affected:** `J/AbstractWindowSkeleton.java`, `J/WindowMultiPiston.java`, client-opening boundary in `J/MultiPistonBlock.java`; `R/assets/multipiston/gui/windowmultipiston.xml` only if required by target BlockUI. Add a small real client entry/helper only if necessary for side isolation.
- **Old -> new:** old ResourceLocation/BOWindow/control API -> Identifier and supported target library methods, with a dedicated-server-safe boundary.
- **Changes:** adapt library methods, preserve control IDs/dimensions/color-direction mapping/input defaults and error feedback; verify rotation/mirror interface compatibility in `J/TileEntityMultiPiston.java` if the resolved Structurize contract changed.
- **Risks:** client classes loaded on server, XML incompatibility, keyboard/mouse regressions from upstream SDL changes, rotations reversed, controls visually working but not sending data.
- **Verification commands:** `.\gradlew.bat compileJava --no-configuration-cache` (now require success for the entire Java source set); `.\gradlew.bat runClient`; `.\gradlew.bat runServer`.
- **Exit:** client/server startup and full GUI/network/rotation behavior work with the real libraries. No GUI simplification or feature removal to reach this gate.

### Stage 8 — Migrate resources and restore modern run/datagen behavior

- **Files affected:** add `R/assets/multipiston/items/multipistonblock.json`; update `R/data/multipiston/recipe/multiblock.json`; inspect existing loot/tag/blockstate/model JSONs and change only those required by target codecs. Run/output setup in `build.gradle`; metadata as necessary. Preserve PNG/XML unless verified incompatibility requires an edit.
- **Old -> new:** implicit inventory model lookup, old ingredient JSON, shared `data` run -> explicit item definition, target-validated JSON and MDK client-data run.
- **Changes:** link the item definition to the existing item model; retain its transforms. Translate ingredients to the target-supported ID/tag encoding and remove obsolete ingredient metadata; compare recipe result and loot structure with actual 26.3 examples. Validate reloadable datapack handling. Choose and consistently include `src/generated/resources` if using the MDK output path. No need to create datagen providers just to port three hand-written data files.
- **Risks:** missing inventory model, material/texture resolution warnings, dropped or uncraftable block, recipe/datapack reload errors, duplicate generated/manual resources, misleading success from a data run with no providers.
- **Verification commands:** `.\gradlew.bat processResources --no-configuration-cache`; `.\gradlew.bat runClientData --no-configuration-cache` after checking `tasks --all` confirms that task; `.\gradlew.bat runClient`; in game use `/reload` and verify crafting, `/loot give @s loot multipiston:blocks/multipistonblock`, normal mining and F3+T resource reload.
- **Exit:** recipe, loot, tag, block/item visuals and reload behavior pass. A no-provider datagen run only verifies launch/configuration, not resource correctness.

### Stage 9 — CI, packaging and complete acceptance

- **Files affected:** `.github/workflows/build.yaml`, `.github/workflows/release.yml`, relevant build/publication configuration, PORTING_STATUS.md; focused future test files only where meaningful coverage is added.
- **Old -> new:** Java 21 ng7 reusable workflows and source artifacts -> Java 25 compatible build/release configuration and verified 26.3 artifacts.
- **Changes:** preserve build/prerelease/release behavior while selecting compatible pinned workflows or equivalent local steps. Verify target Minecraft labels, mandatory dependencies, main/sources/Javadoc output, resource expansion and absence of unwanted legacy/shaded classes. Record acceptance evidence and remaining issues.
- **Risks:** local success masked by Maven Local, Gradle-9-incompatible release plugins, wrong dependency metadata, unintended publication, no actual gameplay tests behind a green `test` task.
- **Verification commands:** `.\gradlew.bat clean build --no-configuration-cache`; `.\gradlew.bat build --configuration-cache` twice if configuration-cache support is retained; `jar tf <actual-output-jar>`; `.\gradlew.bat runServer` and `.\gradlew.bat runClient`. If real registered GameTests were added, `.\gradlew.bat runGameTestServer` must pass. With zero tests, that task can fail and is not a useful mandatory acceptance claim.
- **Exit:** clean target build and copied-world/manual acceptance succeed, followed by client and dedicated-server tests using the packaged JAR and actual required mods. Do not mark port complete based on compilation alone; do not publish during validation.

## 9. Verification matrix and difficult areas

These checks are future acceptance work, not tests already performed.

| Scenario | Evidence required |
| --- | --- |
| Startup / registration | Client and dedicated server load without missing class, registry, metadata or dependency errors; creative tab and place/break block work. |
| Interaction / GUI | Empty hand and held item open the same window once; both dropdowns, number fields, confirmation, invalid-number fallback and equal-direction error work. |
| Network | Server owns the final settings; second client sees them; reopen/reconnect has no stale state; matching versions connect without payload failures. |
| Persistence | Source fixture copied before upgrading; compare NBT keys/values after save/restart, chunk unload/reload and mid-motion reload. Missing legacy output direction still has the old fallback. Do not reopen upgraded worlds in the source version. |
| Motion | Test input/output combinations across six axes, straight and turning paths, range 1/default/10 and source boundary behavior, speeds 1/2/3, sustained/toggled/pulsed redstone. Compare timing to source. |
| Obstacles / fluids | Air, solid obstruction, matching/different block types, bedrock, each excluded push reaction, water/lava/waterlogged cases and unloaded-chunk boundaries. No new duplication or loss. |
| Block entities | Ordinary containers remain excluded; compatible Domum Ornamentum blocks retain appearance and custom/components data through repeated moves and reload. |
| Entities | Player, mob and dropped item in destination area move as expected on server and clients without new desync or collision damage differences. |
| Structurize | Blueprint placement with rotation and mirroring preserves configured input/output direction, including UP/DOWN; error sound still plays. |
| Resources | Recipe crafts one block from unchanged materials; loot and pickaxe behavior match source; inventory/held/ground/frame/world models render correctly; no new unresolved materials. `/reload` and F3+T pass. |
| Distribution | Packaged JAR behaves like the development run; CI works without local unpublished artifacts; no unresolved template variables or example identifiers. |

The most difficult work is (1) obtaining a compatible LDTTeam dependency stack, (2) preserving block-entity data while moving blocks under new serialization/lifecycle APIs, (3) retaining exact redstone/neighbor/fluid/entity behavior, and (4) the dependency-owned GUI/network transition through repeated client API rewrites. Build modernization has a smaller code footprint but is a hard prerequisite.

## 10. Recommended implementation order and decision points

**Baseline-build checkpoint: COMPLETED** (`./gradlew.bat build`, JDK 21, BUILD SUCCESSFUL). The next work is **Stage 0: confirm the migration path and target dependency compatibility**, then **1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9**. No source build reproduction remains pending. Source gameplay fixture capture should happen before implementation; it is separate from the completed build checkpoint. Resource-only work from Stage 8 can be prepared earlier after the target toolchain exists, but its acceptance still requires the complete target runtime.

Do not implement thirteen disposable intermediate ports by default. Use the entire official chain to understand changes, then implement the final target subsystem by subsystem. If an API remains ambiguous, compare its vanilla call site in the original 1.21.1 source and the resolved 26.3 source, as recommended by NeoForge. Do not broaden the port into external repository rewrites without separate scope.

Current unknowns to close: exact compatible target library coordinates, their transitive graph, final target BlockEntity and payload overloads, actual resource-codec acceptance, and source-versus-target runtime behavior. No honest fixed effort estimate is possible until the dependency gate is resolved.

## 11. Deliverable and change-control record

- Initially added, and updated by this correction: **PORTING_PLAN.md**, **PORTING_STATUS.md** only. No additional files created by this correction.
- Other existing files changed: **none**. Branch not changed. Implementation status remains **NOT STARTED**.
- Implementation, compilation fixes, feature changes, saves and assets: **not changed**. No stubs or fake compatibility classes introduced.
- Checks performed: static repository/source/resource inspection; official migration-chain and both target MDK comparison; cached dependency/compiled-artifact inspection; Java executable version; branch/working-tree inspection. Resource parsing and final file-scope checks are read-only.
- Completed externally and confirmed by the user: **baseline 1.21.1 build**, `./gradlew.bat build`, **JDK 21**, **BUILD SUCCESSFUL**. Baseline-build checkpoint: **COMPLETED**.
- Not executed by the assistant during documentation work: additional source/target Gradle builds, datagen, GameTests, client/server launch or gameplay tests. This does not leave the manually verified baseline build pending.
- Manual action now: review the dependency gate and planned stage boundaries. Gameplay acceptance belongs to implementation stages above.
- Suggested manual backup / checkpoint: **`Piston-Unlimited_1.21.1_before_26.3_port_plan`**.
