# Installing Oracle GraalVM 25.4 and the Windows toolchain

Checked 2026-10-30 on Windows 11 x64. Sources: https://www.graalvm.org/downloads/ (versions, licence), https://www.graalvm.org/release-notes/ (release list), the downloaded archive itself (version strings).

## Which GraalVM

- **Oracle GraalVM 25.4** is the innovation release `25.4.4.1.1`, based on JDK `25.0.4.1.1`. Free for production under the GraalVM Free Terms and Conditions (GFTC).
- Naming trap: in download URLs the innovation release is `25i4` (`25i3` before it), and plain `25` is the **25.0 LTS line**. An existing "GraalVM 25" install (`GRAALVM_VERSION="25.0.4"` in its `release` file) is the LTS line and is NOT 25.4. Check with `bin/java -version`: 25.4 prints `Oracle GraalVM 25.4.4.1.1+1.1`.
- StaticFX is confirmed against 25.4 (see `staticfx.md`), which is why 25.4 and not the LTS.

## Download and verify

The Windows x64 zip, as the downloads page patterns imply (the page itself only shows Linux examples):

```
https://gds.oracle.com/download/graal/25i4/latest/graalvm-jdk-25i4-25_windows-x64_bin.zip
```

It answers with a redirect to the pinned build `graalvm-jdk-25i4-25.0.4.1.1_windows-x64_bin.zip` (388,808,349 bytes, 650 files, root folder `graalvm-25.4.4.1.1+1.1`). Use `curl.exe -fL --retry 20 --retry-all-errors -C -` (resumes; the link dropped twice during the first download).

- **No published checksum could be retrieved.** The documented rule "append `.sha256` to the download URL" returned HTTP 404 for the Windows zip and for the Linux example on the downloads page, on every spelling tried (`gds.oracle.com` and `download.oracle.com`, `latest` and `archive` paths). `download.oracle.com/graalvm/25i4/...` is 404 for the zip itself; only `gds.oracle.com` serves 25i4. So: run `7z t` on the archive (it passed), compute the SHA-256 yourself and keep it as a reference, and re-try the `.sha256` URL once in case Oracle publishes it later.
- Reference hash of the archive downloaded on 2026-10-30, computed locally and NOT compared against any Oracle-published value: `CF71435F04D67267563C40572C29BBEF017242DBB996A02BBE95A2E5AD5D761D`.

Extract into its own new empty folder, then rename, e.g. `<apps>/jdk/graalvm-25.4`. `bin/native-image.cmd --version` should print `native-image 25.0.4.1.1 2026-08-18`.

## Do not set JAVA_HOME / GRAALVM_HOME / PATH globally

Call it by full path, or set the three variables inside one build script (next section). Reasons that bit in practice:

- the `native-maven-plugin` takes **`GRAALVM_HOME` before `JAVA_HOME`**, so an older global `GRAALVM_HOME` silently wins over the JDK you put on `PATH`;
- other tools of the machine rely on the global JDK.

## The Windows C toolchain

`native-image` needs the MSVC compiler and linker (`cl.exe`, `link.exe`) plus the Windows SDK headers and libraries, with `INCLUDE` and `LIB` set. Either the "x64 Native Tools Command Prompt" of Visual Studio Build Tools, or a script that sets the same variables. Verified combination: MSVC `14.44.35207` and Windows SDK `10.0.26100.0` (Build Tools 2022). Needs no Visual Studio IDE. Set it up per build process, never globally.

A build script that does all of it for one process (`build-native.cmd <project-folder> [maven args]`):

```bat
@echo off
set TRIAL=%~1
set REST=
for /f "tokens=1,* delims= " %%a in ("%*") do set REST=%%b
call <vsbt>\devcmd.bat
set GRAALVM_HOME=<apps>\jdk\graalvm-25.4
set JAVA_HOME=%GRAALVM_HOME%
set PATH=%GRAALVM_HOME%\bin;%PATH%
cd /d "%~dp0%TRIAL%" || exit /b 1
call mvn.cmd -B -Pnative package %REST%
```

`<vsbt>\devcmd.bat` stands for whatever script sets `INCLUDE`, `LIB` and puts the MSVC `bin` on `PATH`. The `for /f` line is deliberate: in a batch file `=` splits arguments, so `-Dnative.optimization=b` passed through `%1 %2` arrives as two pieces and Maven reads `b` as a lifecycle phase ("Unknown lifecycle phase b").

Call it from PowerShell by full path: `cmd /c "<full path>\build-native.cmd project > build.log 2>&1"`. From Git Bash, `cmd.exe //c "build-native.cmd x"` does not find a script in the current directory.

## Maven

- `org.graalvm.buildtools:native-maven-plugin`: latest on Central `1.1.18` (2026-10-29). The StaticFX examples are pinned to `1.0.0` and `1.1.18` works with 25.4 (see `reachability-annotations.md` and `staticfx.md` for the verified builds).
- Wire it in a `native` profile so a normal `mvn package` stays a plain jar build; `compile-no-fork` bound to `package` builds the executable from the project's own classes and dependencies:

```xml
<profile>
  <id>native</id>
  <build><plugins><plugin>
    <groupId>org.graalvm.buildtools</groupId>
    <artifactId>native-maven-plugin</artifactId>
    <version>${native.plugin.version}</version>
    <executions><execution>
      <id>build-native</id><phase>package</phase>
      <goals><goal>compile-no-fork</goal></goals>
    </execution></executions>
    <configuration>
      <imageName>app</imageName>
      <mainClass>${main.class}</mainClass>
      <metadataRepository><enabled>false</enabled></metadataRepository>
      <buildArgs>
        <buildArg>-O${native.optimization}</buildArg>
        <buildArg>--exact-reachability-metadata</buildArg>
      </buildArgs>
    </configuration>
  </plugin></plugins></build>
</profile>
```

`<native.optimization>2</native.optimization>` as a property lets `-Dnative.optimization=b` select the quick-build mode while iterating. Do not add `<fallback>false</fallback>`: fallback images were removed in GraalVM 25.1, so the plugin only warns that the parameter is deprecated. `<metadataRepository><enabled>false</enabled></metadataRepository>` keeps the build from fetching the community metadata repository (fine for projects with no third-party reflective libraries; leave it enabled otherwise).

## Build cost to expect

Small CLI, default serial GC, 15 GB machine under other load: 3m18s for the first `-O2` build, 2m44s for a repeat, 1m50s with `-Ob`; peak RSS about 2 GiB. Builds are CPU- and RAM-heavy: do not run two at once, and do not time startup while one runs.
