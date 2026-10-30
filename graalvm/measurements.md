# Measured: native image against the JVM, and what each build costs

Measured 2026-10-30, Windows 11 x64, 16 logical CPUs, 15 GB RAM, Oracle GraalVM 25.4.4.1.1 (serial GC, the default), JDK 25 for the JVM baseline. The machine was NOT idle (a video call and other applications were running; CPU load 60-70%). Treat the BEST column as the robust figure and the ratios as the result; absolute milliseconds will differ on another machine. One run under visibly heavier load gave 4.7 s / 2.9 s / 0.55 s for the JavaFX app, the same ordering with worse JVM numbers.

## Method

A script launches the command N times through .NET `Process` with a stopwatch, output discarded, after warm-up launches, and reports the median and the best wall-clock milliseconds (process start to process exit). Both programs are run to completion by themselves: the CLI prints its result, the GUI app has a `--exit` mode that shows the window, fires the button once, checks the result and exits. Compared in the same session, same inputs:

- **jvm**: `java -jar app.jar` / `java -cp <jars> Launcher`.
- **jr + AOT**: the same jar launched by `jr` (the jar launcher in `java.md`) with `-Xjr:aot=true`, cache warm.
- **JDK AOT**: `java -XX:AOTCache=<file>` with a cache created by `-XX:AOTCacheOutput=<file>` from one training run. The cache must be built from a classpath of JARS (a directory on the classpath fails, see `traps.md`).
- **native**: the image built as in `install-and-toolchain.md` (`-O2`, or `-Ob` where stated).

## Trial 1: a small CLI that scans a folder tree and calls a Win32 function through FFM

Input: a tree of 40 folders and 401 files with one reserved-name file, `--dry-run` (scan only).

```
launch                              median      best
jvm      java -jar                  220-268 ms  205-222 ms
jvm      -XX:AOTCache (direct)      204-257 ms  182-194 ms
jr       -Xjr:aot=true              259-303 ms  223 ms
native   -O2                         70-83 ms    64 ms
native   -Ob                         67 ms       60 ms
```

Two rounds each (the median spread shows the noise). Native is about 3.2 times faster than the JVM and about 3.5 times faster than jr with its AOT cache. For a program this small the JDK AOT cache gains about 10% and jr's launch overhead cancels it: do not expect AOT to rescue a tiny CLI; native does.

Sizes: jar 11 KB; its AOT cache 10.5 MB (plus a JRE of about 360 MB on disk); native `-O2` 14.73 MiB, `-Ob` 11.94 MiB (the quick mode was smaller here; it also compiles faster, 1m50s against 2m44s to 3m18s). Peak RSS of the builder about 2 GiB.

## Trial 2: JavaFX 27 window (FXML view, CSS, controller) through StaticFX

`--exit` mode: launch, show the window, fire the button, check the label, exit.

```
launch                              median       best
jvm      jars on the classpath      3.7-3.8 s    3.1-3.3 s
jvm      + JDK AOT cache            2.2-2.7 s    2.0-2.25 s
native   -O2 (StaticFX)             0.53-0.69 s  0.45-0.50 s
```

Native is about 7 times faster than the plain JVM and 4 to 5 times faster than the JVM with an AOT cache. The plain window without FXML (an earlier `-Ob` build) gave 3.2 s / 2.1 s / 0.50 s.

Sizes: native `-O2` 47.48 MiB (34.62 MiB for the plain window, `-Ob`; 37.12 MiB for the FXML view, `-Ob`); the JavaFX windows jars are 9.3 MB, the app jar 8 KB, the AOT cache 31 MB, and the JVM itself about 360 MB (the JDK folder). Build: 5m36s at `-O2` (4m16s in an earlier `-O2` run), 2m31s at `-Ob`; the builder reserved 13 GiB.

The upstream numbers for a larger app (a desktop sampler on a Raspberry Pi 5): time to first window 3.6 s on the JVM (jlink) against 0.5 s native, private memory after all pages 1424 MB against 634 MB.

## Trial 3: the annotation recipes

`reflect` demo (class, resource, bundle by computed name): `-Ob` build 1m00s, 9.66 MiB; it only exists to prove the metadata loop, no timing.

## What this says about when to use it

- Short-lived programs that agents call many times (CLIs): native starts in tens of milliseconds where the JVM needs two hundred; worth it when the call count is large. Cost: a 3-minute build per change, a metadata loop, and one executable per platform.
- A JavaFX app that must come up fast: native is clearly faster to first window, at 47 MiB per executable and 5 minutes per build. Native-image does not run on the JVM's JIT, so peak throughput of a long-running compute-heavy program can be lower; this was not measured.
- Not measured: memory at run time for the trials (upstream reports about half for the large app), and peak throughput.
