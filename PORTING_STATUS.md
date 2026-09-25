# Porting status

Updated: 2026-09-26.

- Source build status: BUILD SUCCESSFUL
- Baseline-build checkpoint: COMPLETED
- Baseline verification: manually verified by the user with `./gradlew.bat build` under JDK 21; result: BUILD SUCCESSFUL. No baseline rebuild is pending.
- Source Minecraft version: 1.21.1
- Target Minecraft version: 26.3
- Current branch: port/26.3
- Implementation status: BUILD-SYSTEM STAGE COMPLETED; JAVA/API MIGRATION NOT STARTED
- Overall port: IN PROGRESS; not playable or release-ready
- Gradle version: 9.2.1
- NeoGradle userdev plugin: 7.1.39
- Java target toolchain: 25
- Java launcher/daemon/compiler installation: `C:\Program Files\Java\jdk-25.0.4`
- `java -version`: 25.0.4; `javac -version`: 25.0.4
- NeoForge version: 26.3.0.10-beta
- NeoForm version: 26.3-1
- Target build command: `./gradlew.bat build`
- Target build result: BUILD FAILED at `:compileJava`; **35 Java compilation errors**; exit code 1
- Error category: **C — Java compilation/API migration errors**
- Gradle configuration: SUCCEEDED
- Minecraft 26.3 / NeoForge dependencies: RESOLVED; game sources prepared and recompiled successfully
- Runtime dependency graph: RESOLVED, with no FAILED dependency entries; this does not establish runtime compatibility of the retained 1.21.1 libraries
- GitHub Actions build/prerelease/release workflows: still depend on `ldtteam/OperaPublicaCreator` reusable workflows **@ng7**; their Java runtime selection was updated to **Java 25**.
- CI build compatibility: **UNVERIFIED**.
- Release/publishing compatibility: **UNVERIFIED** and **must not be treated as production-ready**.
- BlockUI and Structurize: still resolve their old Minecraft **1.21.1** artifacts only as a **temporary build-stage condition**; they are **NOT considered compatible with Minecraft 26.3**.

## Build-system decision

The former one-line `build.gradle` delegated the build to the remote [LDTTeam OperaPublicaCreator ng7/mod.gradle](https://raw.githubusercontent.com/ldtteam/OperaPublicaCreator/ng7/gradle/mod.gradle). That script hardcodes NeoGradle userdev `7.0.+`, which cannot select the 7.1 toolchain required by the modern target, and configures the old data run/Parchment infrastructure. The inspected `main` branch instead uses ForgeGradle 6.0.36. Neither is an appropriate unchanged 26.3 build setup.

Replaced the remote apply with an explicit local build following the official [NeoForge 26.3 NeoGradle MDK](https://github.com/NeoForgeMDKs/MDK-26.3-NeoGradle) and [NeoForge migration toolchain guidance](https://neoforged.net/news/26.1release/). No remote script was patched and no workaround for Java API errors was added.

The local configuration retains the mod ID/group/version-suffix convention, current project URLs/license/authors, direct library dependencies, source/Javadoc artifacts, manifest identity, client/server/GameTest/datagen runs, original datagen output directory, Maven publication metadata/destinations, conditional CurseForge publishing and release-workflow changelog tasks. `runData` delegates to the real modern `runClientData` task. Existing CI workflows now select Java 25. Publication was not executed or certified end-to-end.

Parchment's old 1.21 overlay was removed because 26.3 is unobfuscated. Loader metadata drops the obsolete `modLoader`/`loaderVersion` declarations as in the target MDK; required mods and their BOTH-side declarations are preserved. The generated metadata contains mod version `0.0.1-26.3`, Minecraft range `[26.3]`, and NeoForge minimum `26.3.0.10-beta`.

## Exact files changed

Modified existing files:

1. `build.gradle` — explicit NeoGradle build, Java 25 toolchain, target dependency, runs, resources, packaging and local publishing integration.
2. `settings.gradle` — NeoForge plugin repository, Foojay resolver 1.0.0; remove obsolete mutable start-parameter property injection.
3. `gradle.properties` — Minecraft/NeoForge/Java target versions and ranges; remove old Parchment mapping configuration; retain and label existing library versions.
4. `gradle/wrapper/gradle-wrapper.properties` — Gradle 9.2.1 distribution.
5. `gradle/wrapper/gradle-wrapper.jar` — regenerated Gradle 9.2.1 wrapper.
6. `gradlew` — regenerated wrapper launcher.
7. `gradlew.bat` — regenerated Windows wrapper launcher.
8. `src/main/resources/META-INF/neoforge.mods.toml` — target loader metadata adjustment only.
9. `.github/workflows/build.yaml` — Java 25 for build/prerelease.
10. `.github/workflows/release.yml` — Java 25 for release.
11. `PORTING_STATUS.md` — stage results and next work.

Added files:

12. `gradle/publishing.gradle` — local Maven/CurseForge configuration and changelog tasks formerly supplied by OperaPublicaCreator.
13. `.gitattributes` — wrapper line-ending rules: LF for `gradlew`, CRLF for `gradlew.bat`.

The final hygiene update only added `.gitattributes` and updated this status report. No existing wrapper files were rewritten or renormalized, and no Gradle configuration, dependencies, workflows, mod metadata or Java source files were changed by that update. No Gradle task, API migration, commit or push was performed.

Unchanged: all files under `src/main/java`, gameplay logic, BlockUI/Structurize source and declared dependency versions, `gradle/dependencies.gradle`, access transformers, gameplay/art/data resources, and PORTING_PLAN.md. No features were removed, no compatibility stubs added, and no Java compiler errors suppressed. No commit or push was performed.

Gradle generated its normal ignored outputs/cache files. Diagnostic logs are in `build/port-26.3-build.log` and `build/port-26.3-dependencies.log`; these are not tracked source/configuration additions.

## Verification performed

Before each Gradle invocation the PowerShell process selected:

```powershell
$env:JAVA_HOME = 'C:\Program Files\Java\jdk-25.0.4'
$env:Path = "$env:JAVA_HOME\bin;$env:Path"
java -version
javac -version
```

Both commands reported Java 25.0.4. No machine-specific JDK path was hardcoded into the repository configuration.

| Command/check | Result |
| --- | --- |
| `./gradlew.bat --version` | Gradle 9.2.1; launcher JVM 25.0.4 and daemon JVM at the specified JDK 25 path. Rechecked successfully after wrapper regeneration. |
| `./gradlew.bat wrapper javaToolchains` | Gradle tasks succeeded; Java 25 installation detected and wrapper files regenerated. Updating the running old batch launcher produced a trailing shell `scope` error; subsequent invocation of the regenerated launcher passed. |
| `./gradlew.bat build` | Configuration succeeded, Minecraft/NeoForge preparation completed, then project `:compileJava` failed with 35 errors. No source fixes attempted. |
| `./gradlew.bat dependencies --configuration runtimeClasspath generatePomFileForMavenJavaPublication` | BUILD SUCCESSFUL; runtime graph resolved and local Maven POM generated. No publication performed. |
| Generated mod metadata | Correct target ranges and version; existing required library declarations retained. |
| Source/file-scope and whitespace checks | No Java source changes; modifications limited to build/configuration files and this status report. |

The build log repeats some compiler diagnostics in Gradle's failure summary. **35** is javac's reported count from the original compile task, not the number of matching lines across the repeated log. Warnings from compiling Minecraft/NeoForge itself are separate from the mod's errors.

An initial configuration error while translating publication metadata (`contributor.id`) was corrected in the Gradle script. The final stopping point is C, with no remaining observed A (configuration) or B (dependency resolution) failure.

## First 20 unique compiler issues, grouped by API area

Paths below are relative to `src/main/java/com/ldtteam/multipiston/`. Numbers identify first occurrence after collapsing repeated uses of the same missing symbol/API operation. Override-signature errors and failing superclass calls remain separate diagnostics. The list intentionally does not contain fixes.

| First occurrence | API area | First source location | Diagnostic |
| --- | --- | --- | --- |
| 1 | Resource identifiers / GUI | `AbstractWindowSkeleton.java:7` | `ResourceLocation` cannot be found; repeated at its use on line 28. |
| 2 | Block interaction | `MultiPistonBlock.java:7` | `ItemInteractionResult` cannot be found; also used in return type and SUCCESS result. |
| 3 | Block entity registration | `ModTileEntities.java:17` | `BlockEntityType.Builder` cannot be found. |
| 4 | Block/level access | `MultiPistonBlock.java:57` | `Level.isClientSide` has private access; four occurrences across block and block entity. |
| 5 | Block callbacks | `MultiPistonBlock.java:88` | Old `neighborChanged` declaration no longer overrides a superclass method. |
| 6 | Block codec | `MultiPistonBlock.java:116` | `codec()` no longer overrides a superclass method. |
| 7 | Piston push reaction | `TileEntityMultiPiston.java:187` | `PushReaction.IGNORE` cannot be found. |
| 8 | Piston push reaction | `TileEntityMultiPiston.java:188` | `PushReaction.DESTROY` cannot be found. |
| 9 | Piston push reaction | `TileEntityMultiPiston.java:189` | `PushReaction.BLOCK` cannot be found. |
| 10 | Neighbor updates | `TileEntityMultiPiston.java:217` | `BlockPos` cannot be converted to `Orientation`. |
| 13 | Direction selection | `TileEntityMultiPiston.java:251` | No suitable `Direction.getNearest(int,int,int)` overload; target overload includes a fallback direction. |
| 11 | Block entity serialization | `TileEntityMultiPiston.java:224` | `saveWithId` receives registry access instead of `ValueOutput`; same API issue appears at line 433. |
| 12 | Block entity serialization | `TileEntityMultiPiston.java:228` | `loadWithComponents` expects `ValueInput`, not `CompoundTag, RegistryAccess`. |
| 14 | Block entity serialization | `TileEntityMultiPiston.java:373` | Old `loadAdditional` declaration does not override a superclass method. |
| 15 | Block entity serialization | `TileEntityMultiPiston.java:376` | `super.loadAdditional` expects `ValueInput`, not `CompoundTag, Provider`. |
| 16 | NBT getters | `TileEntityMultiPiston.java:378` | `Optional<Integer>` cannot be converted to `int`; five occurrences. |
| 17 | NBT getters | `TileEntityMultiPiston.java:381` | `Optional<Boolean>` cannot be converted to `boolean`. |
| 18 | NBT key access | `TileEntityMultiPiston.java:382` | `CompoundTag.getAllKeys()` cannot be found. |
| 19 | Block entity serialization | `TileEntityMultiPiston.java:393` | Old `saveAdditional` declaration does not override a superclass method. |
| 20 | Block entity serialization | `TileEntityMultiPiston.java:396` | `super.saveAdditional` expects `ValueOutput`, not `CompoundTag, Provider`. |

Other diagnostics beyond these first 20 unique issues include `handleUpdateTag`/`onDataPacket` override mismatches and missing `LocalPlayer.displayClientMessage`. No payload/GUI/library compatibility conclusion should be inferred from the compiler not reporting every possible later problem.

## Remaining limitations and next recommended stage

**Next: confirm and pin the compatible 26.3 LDTTeam dependency stack, then port entry-point/registry/block API code in the separate authorized Java migration stage.** This build-system stage deliberately retains:

- BlockUI `1.0.188-1.21.1-snapshot`.
- Structurize `1.0.751-1.21.1-snapshot`.
- Transitive Domum Ornamentum `1.0.203-1.21.1-snapshot` and datagenerators `1.20.4-0.1.57-ALPHA`.

These artifacts resolved only as a **temporary build-stage condition**. In particular, the old Minecraft 1.21.1 BlockUI and Structurize artifacts are **NOT considered compatible with Minecraft 26.3**. They were not excluded, replaced with stubs or republished. No application/library API code has been migrated. Userdev compilation, successful dependency reporting and valid metadata do not constitute a working mod or justify a release.

No client/server/gameplay or publication test was run. GitHub Actions build/prerelease/release still use `ldtteam/OperaPublicaCreator` reusable workflows **@ng7**. Their Java runtime selection was updated to **Java 25**, but those remote workflows were not executed. **CI build compatibility is UNVERIFIED. Release/publishing compatibility is UNVERIFIED and must not be treated as production-ready.** Manual check now: review the build-only diff and the dependency gate; gameplay tests must wait until the Java migration compiles.

Suggested manual backup / checkpoint: `Piston-Unlimited_26.3_build-system_Java25_compile-errors`.
