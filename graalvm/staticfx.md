# JavaFX as a GraalVM native image: StaticFX

Checked 2026-10-30, Windows 11 x64, Oracle GraalVM 25.4.4.1.1, JavaFX 27. Built and run, not just read.

Sources:
- https://github.com/HebiRobotics/jfx-static-feature : README (GPL-2.0 with the Classpath Exception). Keep a local clone (`git clone https://github.com/HebiRobotics/jfx-static-feature`); `jfx-static-examples/example-hellofx` is the minimal project to copy.
- https://ennerf.github.io/2026/09/18/JavaFX-27-Native-Image-on-a-Raspberry-Pi-5.html : the author's write-up (why not Gluon Substrate, traps, numbers).
- Maven Central, group `us.hebi.graalvm`.

## What it is

A GraalVM `Feature` plus the statically linked JavaFX native libraries and about 1,000 reflection, JNI and resource entries for JavaFX itself, so a JavaFX app becomes one native executable with the stock `native-maven-plugin`. Windows, Linux and macOS, Intel and Arm. No source patches; the archives match the official OpenJFX release commit. It is why the advice says Oracle GraalVM 25.4: that is what the author confirms against.

Not covered: `javafx.media` and `javafx.web` are linked dynamically (shared libraries must sit next to the executable); Swing/AWT metadata is yours to supply; Metal/mobile details are in the README. Gluon Substrate is not needed (and per the author is stuck on JavaFX 21 and GraalVM CE 23).

## Versions (verified on Maven Central, 2026-10-30)

| Artifact | Version | Note |
|---|---|---|
| `org.openjfx:javafx-controls` (and `-fxml`) | `27` | NOT `27.0.1`: the static archives are pinned to the exact release commit |
| `us.hebi.graalvm:jfx-static-libs` | `27-2` | pairs with openjfx 27; scope `runtime`; the version string follows `<javafx.version>` (bugfix builds get a `-N` suffix) |
| `us.hebi.graalvm:jfx-static-feature` | `1.1.1` | separately versioned; confirmed with Oracle GraalVM 25.4 |
| `org.graalvm.buildtools:native-maven-plugin` | `1.1.18` | the StaticFX examples use `1.0.0`; both fine on 25.4 |

Matching versions are enforced at build time. Table of confirmed combinations: upstream README, "Versions".

## pom essentials

```xml
<properties>
  <javafx.version>27</javafx.version>
  <jfx.static.version>27-2</jfx.static.version>
  <jfx.static.feature.version>1.1.1</jfx.static.feature.version>
  <main.class>app.Launcher</main.class>
</properties>

<dependencies>
  <dependency><groupId>org.openjfx</groupId><artifactId>javafx-controls</artifactId><version>${javafx.version}</version></dependency>
  <dependency><groupId>org.openjfx</groupId><artifactId>javafx-fxml</artifactId><version>${javafx.version}</version></dependency>  <!-- only if you use FXML -->
  <dependency><groupId>us.hebi.graalvm</groupId><artifactId>jfx-static-feature</artifactId><version>${jfx.static.feature.version}</version></dependency>
  <dependency><groupId>us.hebi.graalvm</groupId><artifactId>jfx-static-libs</artifactId><version>${jfx.static.version}</version><scope>runtime</scope></dependency>
</dependencies>
```

plus the `native` profile with the `native-maven-plugin` from `install-and-toolchain.md` (`imageName`, `mainClass`, `-O…`, `--exact-reachability-metadata`). The feature and its metadata are picked up from the classpath: no extra build args are needed. Build: `build-native.cmd <project> ` (= `mvn -Pnative package` inside the MSVC and GraalVM environment). StaticFX's own `-Pnative` profiles also pass `<metadataRepository><enabled>false</enabled></metadataRepository>`.

- Use a separate `Launcher` class with `main` that calls `Application.launch(App.class, args)`, not the `Application` subclass as the main class (the example does this so the JavaFX launcher does not take the "toolkit not on module path" path).
- Windows: an executable containing an `Application` subclass is built as a GUI-subsystem exe (no console window). `<buildArg>-Djfx.static.gui=false</buildArg>` (or `auto` / `true`) overrides it (documented upstream, not tried here).
- Linux needs the GTK dev packages at build time and `libgtk-3-0 libxtst6 libxxf86vm1 libgl1` at run time; macOS needs a launcher that gives the first thread to Cocoa for `Platform.startup()` (upstream README). Neither was tried here; the machine is Windows.

## Your own views: the reachability annotations

StaticFX's metadata covers JavaFX only. Reflectively loaded application code (FXML controllers, CSS-named skins) needs `@ReachableFxView` / `@ReachableFxResources` from `reachability-annotations`; the recipe, the generated output and the FXML traps (explicit imports, `javafx-fxml`, `.bss`) are in `reachability-annotations.md`.

## What was built and what it did

1. **Plain window** (a `Label` and a `Button`, no FXML): StaticFX alone, no annotations, `--exact-reachability-metadata`, `-Ob`: built first time, 34.62 MiB, in 2m31s. Ran; the window rendered "Hello JavaFX 27+30" with the Modena button, painted into a bitmap with `PrintWindow`.
2. **The same app with an FXML view, a CSS file and a controller with an `@FXML` field and handler**, `@ReachableFxView` on the loader class: generated metadata for the controller, the node types and the resources. First run crashed at load on the wildcard-import probe (see `traps.md`); with explicit imports in the FXML it passed an in-process self-test (`--exit`: show, fire the button, check the label, exit 0 or 3).
3. Measurements: `measurements.md`.

## Traps specific to StaticFX

- The versions of `javafx-*` and `jfx-static-libs` must be the same release (no `27.0.1` with `27`).
- Without a console the exception is invisible: run `cmd /c "app.exe > run.log 2>&1"` (`traps.md`).
- Missing JNI metadata can crash with a segmentation fault instead of an exception; missing rendering resources can silently produce artifacts. Use exact mode and run every view.
- Agent-generated (tracing agent) metadata goes stale; the annotations regenerate it on every compile.
- The `Application` subclass's no-argument constructor must be reachable; the feature registers it for reachable `Application` subclasses.
