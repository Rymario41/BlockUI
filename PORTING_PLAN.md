# BlockUI port to Minecraft 26.3

## Scope and preserved base

- Branch: `port/26.3`.
- Functional base: official `upstream/port/26`, revision `bc14ca5688c220f10fc93e7f759cde941fee994b`.
- Source target: Minecraft 26.1.2, NeoForge 26.1.2.68-beta, Java 25.
- This is an incremental update of the official 26.x port, not a restart from Minecraft 1.21.1.
- This stage changes build/configuration only. All 107 main Java files, existing test sources, gameplay/UI functionality and access transformer rules remain intact. No compatibility classes, excluded Java files or ignored compilation failures are introduced.

## Baseline checkpoint: recorded

The user already ran `./gradlew.bat build` on the original build setup. It failed during project evaluation at `build.gradle:1`, before Java compilation (category A):

```text
Missing implementation of resolved method
'abstract java.lang.String getDisablePropertyName()'
of abstract class
org.codehaus.groovy.transform.stc.AbstractExtensionMethodCache
```

The original wrapper was Gradle 9.4.0. Reproducing this known baseline failure is not unfinished work.

## Build-system decision

The original `build.gradle` delegated all build logic to the remote LDTTeam OperaPublicaCreator `ng7/gradle/mod.gradle`. That script selects NeoGradle `7.0.+` and several older build plugins. It does not select the toolchain used by the official 26.3 MDK and already fails evaluation in this repository. The exact transitive plugin responsible for the Groovy linkage failure has not been isolated; replacing the incompatible shared setup avoids relying on it.

Replace the remote Gradle script with local build configuration based on the official NeoForge 26.3 NeoGradle MDK:

| Component | Original | Target |
| --- | --- | --- |
| Minecraft | 26.1.2 | 26.3 |
| NeoForge | 26.1.2.68-beta | 26.3.0.10-beta |
| NeoGradle | Remote `7.0.+` selection | Pinned 7.1.39 |
| Gradle wrapper | 9.4.0 | 9.2.1, matching the MDK |
| Java toolchain | 25 | 25; local JDK 25.0.4 |

Local facilities include project identity/versioning, metadata expansion, access transformers, client/server/GameTest/client datagen runs, generated resources, JUnit 4 tests, source/Javadoc archives, Maven publication, conditional CurseForge publication and release changelog tasks. Optional facilities unused by this repository are not copied wholesale from the shared script.

JUnit is explicitly declared as 4.13.2 for the existing JUnit 4 tests. JetBrains annotations retain the existing direct declaration, 21.0.1; platform dependency conflict resolution may select a newer version. CurseForgeGradle is pinned to 1.3.33. No old Minecraft artifacts or replacement API dependencies are added to conceal source incompatibilities.

## Stages and recommended order

### 0. Confirm target tooling and migration path

- Files affected: documentation only.
- Change: compare the current configuration and remote script with the official MDK; confirm Java and dependency versions.
- Old behavior: remote build logic selects a mutable 7.0.x NeoGradle version and fails evaluation.
- Expected behavior: pinned 26.3 MDK toolchain, with Java 25 selected before Gradle starts.
- Risk: the MDK and remote branches are mutable references; record exact versions locally.
- Verification: `java -version`, `javac -version`, MDK inspection and later dependency resolution.

### 1. Replace the required build facilities locally (current stage)

- Files affected: `build.gradle`, `settings.gradle`, `gradle.properties`, wrapper files as changed by regeneration, `gradle/publishing.gradle`, `src/main/resources/META-INF/neoforge.mods.toml`, `.github/workflows/build.yaml`, `.github/workflows/release.yml`, and porting documents.
- Change: use the plugins DSL, NeoForge plugin repository and Foojay resolver 1.0.0; pin the target versions; retain Java 25; regenerate the wrapper; implement the used build/publication facilities locally.
- Old behavior: 26.1.2 dependency/ranges and remote configuration; workflow inputs still selected Java 21 despite the source Java 25 target.
- Expected behavior: resolve Minecraft 26.3 and NeoForge, expand target metadata and reach `compileJava` using Java 25. Client data generation replaces the older launch type, with `runData` delegating to `runClientData` and preserving its output directory. Metadata follows the MDK without legacy `modLoader`/`loaderVersion` fields.
- Risks: upstream access transformers or Java APIs may refer to changed members; publication behavior needs separate end-to-end validation; remote reusable workflows remain an external dependency.
- Verification: `./gradlew.bat --version`, `./gradlew.bat wrapper javaToolchains`, `./gradlew.bat build`; local-only `./gradlew.bat generatePomFileForMavenJavaPublication dependencies --configuration runtimeClasspath`.
- Acceptance: SUCCESS or C (Java/API compilation errors). Configuration/dependency failures A/B do not satisfy this stage. Runtime and publishing success cannot be inferred from reaching compilation.

### 2. Migrate remaining Java APIs only after separate authorization

- Files affected: only source files identified by compiler diagnostics; access transformer rules if their corresponding members changed.
- Change: address differences from 26.1.2 through 26.2 to 26.3 while preserving the upstream implementation and functionality.
- Old behavior: existing 26.1.2 rendering, GUI, input, texture and utility API usage.
- Expected behavior: equivalent behavior using the actual target APIs; no stubs or feature removal.
- Risks: 26.2 rendering/GUI changes and 26.3 input/rendering changes can require coordinated changes; compilation alone does not verify visual behavior.
- Verification: `./gradlew.bat compileJava`, then `./gradlew.bat build` without exclusions or failure suppression.
- Status: NOT STARTED in this stage.

### 3. Validate runtime, datagen and release integration

- Files affected: only configuration/resources needing demonstrated corrections; workflow changes only after inspecting the reusable workflow contract.
- Change: validate the existing UI/test behavior and generated data, then the publication/CI contract.
- Old behavior: workflows delegate build/prerelease/publish to OperaPublicaCreator `@ng7`.
- Expected behavior: a verified Java 25 CI build and reviewed publication artifacts for 26.3, preserving project identity, license, sources and Javadoc.
- Risks: a locally generated POM does not verify authenticated publishing, archive contents, remote workflow behavior or a functioning Minecraft client. Beta NeoForge APIs may change.
- Verification: `./gradlew.bat build`, `./gradlew.bat runClient`, applicable server/GameTest runs, `./gradlew.bat runData`, artifact inspection, and separately authorized CI/publication validation. Do not upload releases during this stage.
- Status: NOT STARTED. CI and release/publishing compatibility remain UNVERIFIED; not production-ready.

## Environment and reference material

Use `C:\Program Files\Java\jdk-25.0.4` for `JAVA_HOME`, with its `bin` first on `PATH`. Check both Java executables before Gradle, and stop if Gradle runs on a different Java major version.

References inspected for this stage:

- [Official NeoForge 26.3 NeoGradle MDK](https://github.com/NeoForgeMDKs/MDK-26.3-NeoGradle): build, settings, properties, wrapper and metadata.
- [OperaPublicaCreator ng7 mod.gradle](https://github.com/ldtteam/OperaPublicaCreator/blob/ng7/gradle/mod.gradle): original remote build facilities and plugin selection.
- [NeoForge 26.1 release guidance](https://neoforged.net/news/26.1release/): Java 25 and modern Gradle tooling baseline.
- [26.1.x to 26.2 migration primer](https://docs.neoforged.net/primer/docs/26.2/): remaining rendering/GUI changes.
- [26.2 to 26.3 migration primer](https://docs.neoforged.net/primer/docs/26.3/): remaining input/rendering changes.

The acceptance evidence and exact resulting file list are recorded in `PORTING_STATUS.md`.
