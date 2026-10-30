# GraalVM native image / AOT

Read before building any Java program ahead of time with GraalVM (native image, or AOT for a CLI that agents call many times).

## The instruction

The advice we are following, as it was given:

> *Given that you use a lot of agents, I'd also try AOT with Oracle GraalVM (25.4). Agents should be able to figure out the configuration and metadata by themselves after pointing them at the reachability-annotations recipes and StaticFX documentation.*

So: Oracle GraalVM 25.4; the configuration and the reachability metadata are worked out by the agent itself, from the reachability-annotations recipes (for plain Java, including the reflection-free paths) and the StaticFX documentation (for JavaFX).

## Learn once, then keep what you learnt here

Doing this the first time means downloads and a lot of reading (the GraalVM release, the reachability-annotations recipes, the StaticFX docs, trial builds). **Do not re-learn it every time.** Whoever does GraalVM work must leave what they learnt in this folder, organised, so the next run reads it instead of starting over:

- `graalvm.md` (this file): the short, current rules. What to install and from where (exact versions), the build commands, the pom configuration, the traps. Replace a rule when it changes; never keep two competing versions.
- `graalvm/` beside it: one note per topic (installing and the toolchain, reachability annotations with worked recipes, StaticFX, measured startup and size, traps), each with the source links and the date it was checked.
- Read this file and `graalvm/` FIRST. Go back to the upstream documentation only for what is not here, or when the versions here are out of date, and then update the notes.

Everything here is public: no personal paths, no private project or organisation names (see `../guidelines-coding/git-secrets.md`).

## Current rules

Written 2026-10-30 after the first learning run: three trial builds, built and run on Windows 11 x64. Detail, sources and the date each was checked are in `graalvm/` (read the note for your topic before you start). Linux and macOS were not tried.

**Versions that were verified to work together**

- Oracle GraalVM **25.4.4.1.1** (JDK 25.0.4.1.1), free under the GFTC. Download from `gds.oracle.com`, path `25i4` (`25` is the LTS line, not 25.4). No Oracle checksum was retrievable: `7z t` the archive and record your own hash. `graalvm/install-and-toolchain.md`.
- `org.graalvm.buildtools:native-maven-plugin` `1.1.18`; `us.hebi.graalvm:reachability-annotations` and `reachability-processor` `1.0.0-RC5`; for JavaFX `org.openjfx` `27` with `us.hebi.graalvm:jfx-static-libs` `27-2` and `jfx-static-feature` `1.1.1`. Re-check Maven Central before relying on newer ones.

**Rules**

1. **Never set JAVA_HOME, GRAALVM_HOME or PATH globally for GraalVM.** Set them inside one build script, per process. `GRAALVM_HOME` is read before `JAVA_HOME` by the Maven plugin, so an older global value silently wins. On Windows the same script sets up the MSVC environment (`INCLUDE`, `LIB`, `PATH`).
2. **Keep native builds in a `native` Maven profile** (`compile-no-fork` bound to `package`), so `mvn package` stays a plain jar build. Do not add `<fallback>` (removed in 25.1). Iterate with `-Dnative.optimization=b` (about half the build time), measure and ship with `-O2`. One build at a time: the builder reserves up to 85% of RAM.
3. **Always build with `--exact-reachability-metadata`** and run every code path. A lookup without metadata then fails loudly with the entry to add. A green build proves nothing; a missing registration shows up only when the code runs.
4. **Let the agent work out the metadata with the loop in `graalvm/reachability-annotations.md`:** build, run, read the `Missing*RegistrationError`, put ONE `@Reachable` / `@ReachableMember` / `@ReachableFxView` on the class that performs the lookup (not on the target), rebuild, READ the generated JSON in `META-INF/native-image/reachability-generated/<project>/`. Use `annotationProcessorPaths` (JDK 23+) and set `-Areachability.outputFormat` explicitly. The tracing agent finds candidates; it does not replace the annotations.
5. **The annotations do not cover FFM.** Hand-write the `foreign` section (`downcalls` of `{returnType, parameterTypes}`) in `src/main/resources/META-INF/native-image/<groupId>/<artifactId>/reachability-metadata.json`; the runtime error prints the exact entry, and every distinct function signature needs one.
6. **JavaFX: StaticFX plus the annotations, nothing else.** Dependencies `org.openjfx:javafx-controls` (and `javafx-fxml` if you load FXML; it does not come with controls), `jfx-static-feature`, `jfx-static-libs` (scope `runtime`), the same release number for openjfx and the static libs. A separate `Launcher` class with `main`. **Use explicit imports in FXML** (`<?import javafx.scene.layout.VBox?>`, not `javafx.scene.layout.*`): in exact mode `FXMLLoader`'s probes of wildcard packages are fatal. `@ReachableFxView` on the class that loads the FXML. `graalvm/staticfx.md`.
7. **Give a GUI app a `--exit` self-test** (show the stage, fire the button, check, exit 0 or 3); a GUI image has no console, so run it as `cmd /c "app.exe > run.log 2>&1"` to see exceptions. To look at the window use `PrintWindow`, never a screen-rectangle copy (it captures whatever is on top).
8. **Measure before promising.** Launch times on the trial machine: a tiny CLI 64 ms native against 205 ms JVM (jr with an AOT cache gave no gain at this size); a JavaFX window 0.45-0.5 s native against 3.1-3.3 s JVM and 2.0-2.25 s with a JDK AOT cache. Builds cost 2-6 minutes each. `graalvm/measurements.md`.

**Where the rest lives**

- `graalvm/install-and-toolchain.md`: download, verify, layout, the Windows toolchain, the build script, the Maven profile.
- `graalvm/reachability-annotations.md`: the loop, the pom setup, worked recipes (class, resource, bundle, FXML, FFM), the generated output.
- `graalvm/staticfx.md`: StaticFX versions, pom, what was built, specific traps.
- `graalvm/measurements.md`: method, numbers, sizes, build times.
- `graalvm/traps.md`: everything that cost time, in one list.
- `graalvm/trial-sources.md`: the complete sources of the trial programs, to reproduce a build from the notes.

When you learn something new, change the note it belongs to and the rule here, with the date; never keep two versions of a rule.
