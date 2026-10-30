# Blending ArcadeDB's java.util.logging into SLF4J/Logback

This assumes logback-classic is already the SLF4J backend of the app, and addresses a specific problem: even after you've wired up logback-classic and the terse converters correctly, ArcadeDB's own log lines keep showing up in a completely different format (and sometimes writing their own `log/arcadedb.log.*` files on the side). This happened on a JavaFX desktop app and the fix generalizes to any embedded library that ships its own `java.util.logging` bootstrap.

## The problem

ArcadeDB does not use SLF4J internally. It has its own logging facade (`com.arcadedb.log.LogManager`, `com.arcadedb.log.DefaultLogger`) that wraps plain `java.util.logging` (JUL). Worse, `DefaultLogger`'s constructor calls `installCustomFormatter()`, which runs the first time any ArcadeDB code touches its logger (in practice, during `ArcadeDBServer.start()`), and that method loads a properties file named `arcadedb-log.properties` bundled inside `arcadedb-engine.jar`, via `ClassLoader.getResourceAsStream("arcadedb-log.properties")`, and feeds it to `java.util.logging.LogManager.getLogManager().readConfiguration(inputStream)`.

That bundled properties file looks like this (extracted from `arcadedb-engine-25.7.1.jar`):

```properties
handlers = java.util.logging.ConsoleHandler, java.util.logging.FileHandler
.level = INFO
io.undertow.level=WARNING
org.xnio.level=WARNING
org.jboss.level=WARNING
com.arcadedb.level = INFO
java.util.logging.ConsoleHandler.level = INFO
java.util.logging.ConsoleHandler.formatter = com.arcadedb.utility.AnsiLogFormatter
java.util.logging.FileHandler.level = INFO
java.util.logging.FileHandler.pattern=./log/arcadedb.log
java.util.logging.FileHandler.formatter = com.arcadedb.log.LogFormatter
java.util.logging.FileHandler.limit=100000000
java.util.logging.FileHandler.count=10
```

So ArcadeDB installs its own `ConsoleHandler` (with its own `AnsiLogFormatter`, producing lines like `2026-07-07 05:47:39.574 INFO  [ArcadeDBServer] <ArcadeDB_0> message`) and its own rotating `FileHandler` writing `./log/arcadedb.log.0`, `.1`, `.2`, etc — entirely independent of SLF4J/logback, and it does this unconditionally on startup, regardless of any logging setup your own app did first.

## Why the obvious fixes don't fully work

**Setting a JUL level in your own code** (`Logger.getLogger("com.arcadedb").setLevel(Level.WARNING)`) only changes the threshold — ArcadeDB's own `ConsoleHandler`/`FileHandler` are still attached and still format with `AnsiLogFormatter`/`LogFormatter`. This just makes the mismatched format rarer, not gone.

**Installing `SLF4JBridgeHandler` before constructing `ArcadeDBServer`** doesn't survive either. `LogManager.readConfiguration(InputStream)` always calls `reset()` internally before applying the new config — `reset()` strips every handler from every logger it knows about. So whatever you installed beforehand gets wiped the moment ArcadeDB's own `readConfiguration()` call runs during `server.start()`.

**Installing the bridge after `server.start()` returns** does work for everything logged afterwards (which in practice is the vast majority of ArcadeDB's lifetime output: query-planner warnings, index-scan warnings, ongoing server activity), but leaves a handful of startup-banner lines in ArcadeDB's own format — those are logged synchronously inside `.start()`, before your code regains control to reinstall the bridge.

## The actual fix: shadow the properties file on the classpath

`ClassLoader.getResourceAsStream(name)` resolves to whichever entry on the classpath comes first for that resource name. If your run classpath lists your own compiled resources before the library's jar — e.g. `java -cp "target\classes;lib\*" ...`, where `target\classes` (built from `src/main/resources`) precedes `lib\*.jar` — then simply placing a file at `src/main/resources/arcadedb-log.properties` makes your copy win over the one bundled inside `arcadedb-engine.jar`. ArcadeDB's own bootstrap code ends up loading *your* config without knowing the difference.

That means you can tell ArcadeDB's own `readConfiguration()` call to install the SLF4J bridge handler itself, declaratively, instead of a `ConsoleHandler`:

```properties
# src/main/resources/arcadedb-log.properties
# Shadows the arcadedb-log.properties bundled inside arcadedb-engine.jar.
handlers = org.slf4j.bridge.SLF4JBridgeHandler
.level = INFO
```

A `java.util.logging` properties file's `handlers` key accepts the fully-qualified name of any class with a public no-arg constructor — JUL instantiates it and attaches it to the root logger for you. `org.slf4j.bridge.SLF4JBridgeHandler` (from the `org.slf4j:jul-to-slf4j` artifact) qualifies, so this one line is all it takes. Because this now happens as *part of* ArcadeDB's own config load (not a separate step racing against it), the bridge is active from the very first line ArcadeDB logs — no startup-banner gap, no leftover `ConsoleHandler`, no separate `arcadedb.log.*` files on disk, and no duplicate lines.

## Required dependency

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>jul-to-slf4j</artifactId>
    <version>${slf4j.version}</version>
</dependency>
```

Use the same version as `slf4j-api` — `jul-to-slf4j` is published in lockstep with the rest of the SLF4J release train.

## Defensive fallback in code

Classpath shadowing is implicit and can break if the classpath layout ever changes (most notably: packaging into a shaded/uber-jar, where two jars providing a resource of the same name become ambiguous unless a resource transformer picks the right one). So it's worth keeping a small code-level fallback that reasserts the bridge if the properties-file trick didn't take effect, called right after `ArcadeDBServer.start()` returns:

```java
server = new ArcadeDBServer(cfg);
server.start();
ensureArcadeDbLoggingBridged();   // normally a no-op; see below
...

/**
 * Normally a no-op — src/main/resources/arcadedb-log.properties already made ArcadeDB install
 * SLF4JBridgeHandler itself. This only fires if that shadowing didn't take effect.
 */
static void ensureArcadeDbLoggingBridged() {
    if (!org.slf4j.bridge.SLF4JBridgeHandler.isInstalled()) {
        java.util.logging.LogManager.getLogManager().reset();
        org.slf4j.bridge.SLF4JBridgeHandler.install();
    }
}
```

`SLF4JBridgeHandler.isInstalled()` checks whether an instance is already attached to the root logger, so this never double-installs even if both the properties file and this fallback fire.

## logback.xml side: pick levels once the format is unified

Once ArcadeDB's (and GraalVM/Truffle's, which ArcadeDB pulls in for its scripting engine) log records are flowing through SLF4J, they're ordinary logback loggers by name and can be leveled the normal way:

```xml
<contextListener class="ch.qos.logback.classic.jul.LevelChangePropagator">
    <resetJUL>true</resetJUL>
</contextListener>

<logger name="com.arcadedb" level="INFO"/>
<logger name="com.arcadedb.script.GraalPolyglotEngine" level="OFF"/>
<logger name="com.oracle.truffle" level="ERROR"/>
<logger name="org.graalvm" level="ERROR"/>
```

**Don't over-suppress.** The whole point of this exercise was to get ArcadeDB's output into *your* format — not to hide it. Once the format is unified, ArcadeDB at `INFO` reads just like the rest of your app (its startup banner, HTTP server line, "server started" — all in the terse `HH:mm:ss.S│LEVEL│Class│msg` layout), and those lines are genuinely useful: they tell you the DB actually came up, on which port, from which data directory. A first instinct is to clamp `com.arcadedb` to `WARN` to "keep it quiet", but that throws away exactly the confirmation you want during startup, and you've already paid to bring it into your pipeline — suppressing it now is self-defeating. Keep `com.arcadedb` at `INFO` (matching your root level). Only the truly noisy, non-actionable stuff stays suppressed: `GraalPolyglotEngine` (interpreter-only chatter) at `OFF`, and the GraalVM/Truffle `sun.misc.Unsafe` deprecation warnings at `ERROR`. Reserve `WARN` for `com.arcadedb` only if a specific deployment proves too chatty — and then say so, rather than defaulting to silence.

(The one line that stays outside this pipeline is the JVM's own `WARNING: sun.misc.Unsafe::objectFieldOffset ...` printed directly to stderr by the module system before any logger sees it — that's a launcher-flag concern, e.g. `--sun-misc-unsafe-memory-access=allow`, not a logback level.)

`LevelChangePropagator` with `resetJUL=true` pushes these levels back down into the actual `java.util.logging.Logger` objects, so `isLoggable()` short-circuits on the JUL side for whatever you *do* suppress, instead of constructing a record and handing it to the bridge just to have logback discard it.

## Verifying it worked

- `mvn dependency:tree | grep -i slf4j` shows exactly one SLF4J binding (`logback-classic`) and no competing binding such as `slf4j-jdk14` (ArcadeDB's server module pulls that in too — exclude it separately in `pom.xml`, it's a different problem from this one but easy to conflate).
- Run anything that boots `ArcadeDBServer` and confirm the console shows one consistent format throughout, including ArcadeDB's own INFO startup lines — no raw `2026-07-07 05:47:39.574 INFO  [ArcadeDBServer] ...` lines mixed in (they should now read `05:47:39.5│INFO │ArcadeDBServer│...` in your terse layout).
- Confirm no new `log/arcadedb.log.*` files get written — that side-channel should be fully gone.
- Confirm no duplicate lines for the same event (that would mean two handlers are both live — usually because the shadowed properties file wasn't picked up and both ArcadeDB's own handler and your bridge ended up attached).

## Generalizing beyond ArcadeDB

This pattern applies to any dependency that (a) logs via raw `java.util.logging` and (b) bootstraps its own JUL config from a properties file loaded off the classpath by name. Find the resource — usually visible with `jar tf the-library.jar | grep -i log` or by grepping its logging wrapper class for `getResourceAsStream` — and shadow it in `src/main/resources` with your own version if you control the classpath order. This is generally simpler and more complete than fighting the library's logging setup from your own code after the fact, since your own code can only react *after* the library resets everything.
