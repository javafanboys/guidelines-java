# GraalVM traps hit in practice

Checked 2026-10-30, Oracle GraalVM 25.4.4.1.1, Windows 11 x64. Each item cost real time once. Details and sources are in the sibling notes.

## Toolchain and build

- **`GRAALVM_HOME` beats `JAVA_HOME` and `PATH`.** The `native-maven-plugin` reads `GRAALVM_HOME` first, so an older GraalVM exported globally silently builds your image with the wrong version while `java -version` on `PATH` looks right. Set `GRAALVM_HOME`, `JAVA_HOME` and `PATH` inside the build script, per process (`install-and-toolchain.md`). The plugin's log line "Found GraalVM installation from GRAALVM_HOME variable" names the one it picked: check it.
- **`25` is not `25.4`.** In download URLs `25` is the 25.0 LTS line and `25i4` the 25.4 innovation release. A local "graalvm-25" folder is probably the LTS line.
- **No published checksum for 25i4.** The documented `<url>.sha256` returned 404 for every spelling. Run `7z t`, record a locally computed hash, say in the notes that it is unverified.
- **A batch file splits arguments at `=`.** `build.cmd app -Dnative.optimization=b` reaches Maven as `-Dnative.optimization` and `b`, and Maven reports "Unknown lifecycle phase b". Take the rest of the line with `for /f "tokens=1,* delims= " %%a in ("%*") do set REST=%%b`.
- **`<fallback>false</fallback>` is obsolete.** Fallback images were removed in GraalVM 25.1; the plugin warns that the parameter is deprecated. Leave it out.
- **Builds are heavy and the builder grabs RAM.** With under 8 GiB free it reserves 85% of system memory (13 GiB on a 15 GB machine). A small CLI took 2m44s to 3m18s at `-O2` (1m50s at `-Ob`), a JavaFX app 4m16s at `-O2` (2m31s at `-Ob`). Use `-Dnative.optimization=b` while iterating, `-O2` for the image you measure or ship, one build at a time, and never time startup while a build runs.
- **Native-image on Windows needs MSVC on the build process's `PATH`/`INCLUDE`/`LIB`.** Set up per process (`install-and-toolchain.md`), not globally.
- **From Git Bash, `cmd.exe //c "script.cmd"` does not find a script in the current directory.** Call it by full path, or from PowerShell.

## Metadata

- **Exact mode (`--exact-reachability-metadata`) is the right default for agents, and it also turns harmless misses into hard errors.** A lookup that is meant to fail (an optional class probed with `Class.forName`, a probed file) throws a `Missing*RegistrationError`, which `catch (ClassNotFoundException)` does not catch. Seen with FXML: `<?import javafx.scene.control.*?>` makes `FXMLLoader` try `javafx.scene.control.VBox` first; the processor registers only the class each name resolves to, so the probe killed the app at load with `Cannot reflectively access the class 'javafx.scene.control.VBox'`. **Use explicit imports in FXML** (`<?import javafx.scene.layout.VBox?>`), or register the probes by exact name.
- **Compiling an `@FXML` controller needs `org.openjfx:javafx-fxml`** on the classpath; `javafx-controls` does not bring it.
- **The annotation processor does not cover FFM.** `downcallHandle` without a `foreign` entry fails at the first call, not at build time (`reachability-annotations.md`).
- **Annotation processors only run with `annotationProcessorPaths` on JDK 23+.** A plain dependency on the processor jar silently generates nothing.
- **A green native build proves nothing about metadata.** Missing registrations fail only when the lookup runs. Run every path, and have the app test itself (see below).
- **Generated metadata is not cleaned when the output format changes.** Delete `META-INF/native-image/reachability-generated` first.

## Running and testing

- **A GUI image has no console on Windows.** StaticFX builds a JavaFX `Application` as a GUI-subsystem exe, so stderr and stdout go nowhere and a crash looks like "the window never appeared". Run it as `cmd /c "app.exe > run.log 2>&1"` to see the exception (the 2026-10-30 FXML failure above was found exactly this way), or build with `-Djfx.static.gui=false` (documented by StaticFX, not tried here).
- **Give a GUI app a `--exit` self-test instead of testing by eye.** Show the stage, fire the button programmatically, compare the result, `System.exit(3)` on mismatch, `Platform.exit()` otherwise. It proves reflective handlers work in the image, and it makes the startup measurable by a script.
- **To look at a window, ask the window to paint itself: `PrintWindow(hwnd, hdc, 2)`.** Do NOT copy a screen rectangle. Another window (a video call, a document) can sit on top of your app's position and the capture contains it. That happened once during this work; the capture was deleted unread.
- **A JDK AOT cache cannot be built from a classpath with a directory on it.** `-XX:AOTCacheOutput=...` fails with "Cannot have non-empty directory in paths": put the app in a jar. The failed run then measures a JVM with no cache at all, and looks like "AOT does nothing".
- **Windows PowerShell 5.1 runs on .NET Framework:** `ProcessStartInfo.ArgumentList` does not exist there. Build the `Arguments` string with quotes.
- **For a tiny CLI, a launcher with an AOT cache is no faster than the plain JVM** (measured, `measurements.md`); do not assume it.
- The JavaFX warning "Unsupported JavaFX configuration: classes were loaded from 'unnamed module'" also prints from the native image (classpath build). It is harmless.

## Network

- Oracle's download host and GitHub dropped intermittently (`Could not resolve host`, `early EOF` on a 700 MB clone). Use `curl.exe -fL --retry 20 --retry-all-errors -C -` and re-run failed clones; check the exit status rather than the absence of output (`curl -s` hid a failed download as "no file").
