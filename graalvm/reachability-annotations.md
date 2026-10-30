# Reachability metadata: the annotation recipes, and what they do not cover

Checked 2026-10-30 against Oracle GraalVM 25.4.4.1.1 on Windows x64. Every snippet below was built and run in a trial, not copied from documentation.

Sources:
- https://github.com/HebiRobotics/reachability-annotations : README and `BEST_PRACTICES.md` (the "recipes"; Apache-2.0; `us.hebi.graalvm:reachability-annotations` and `reachability-processor`, version `1.0.0-RC5`, the newest on Maven Central on 2026-10-30). `BEST_PRACTICES.md` is written so an agent can follow it: "Follow BEST_PRACTICES.md to add reachability annotations to module X and verify the generated metadata".
- https://www.graalvm.org/jdk25/reference-manual/native-image/metadata/ : the `reachability-metadata.json` format, conditions, `--exact-reachability-metadata`, the `foreign` section.
- Keep a local clone of the first (`git clone https://github.com/HebiRobotics/reachability-annotations`) and read `BEST_PRACTICES.md` there rather than fetching pages over and over.

## The rule

Only annotate what native-image cannot work out itself: a class or member looked up by a string that is not a constant (`Class.forName(name)`, `getMethod`, JNI `FindClass`), and a resource or bundle loaded by a computed name. `new Foo()`, `Foo.class`, and reflective calls whose arguments are constants in the same method need nothing; an annotation on those only grows the image.

## The loop an agent follows (no tracing agent needed to start)

1. Build with `--exact-reachability-metadata` (GraalVM 23+). A lookup without metadata then fails with a `Missing*RegistrationError` that names the entry, instead of behaving as if the class or resource did not exist.
2. Run every code path (every view, dialog, plugin name). Add `-XX:MissingRegistrationReportingMode=Warn` to the run to list all gaps in one go instead of dying on the first.
3. For each error, put one annotation on the class that performs the lookup (table below). Prefer `classes = X.class` over `classNames` (compile-checked); use `memberAccess` as small as the follow-up call needs.
4. Rebuild, then READ the generated JSON (below). Treat `Skipping invalid pattern` and similar build warnings as errors: invalid entries are dropped without failing the build.
5. Run again. Missing metadata fails only when the lookup runs, so a build that succeeds proves nothing.

The tracing agent (`java -agentlib:native-image-agent=config-output-dir=<dir> -jar app.jar`, run on the GraalVM `java`) is a useful candidate finder: it lists accesses that really happened. Use its output as a list of findings, not as the shipped file: it contains JDK internals, misses code paths that did not run, and goes stale when the code changes.

## pom setup

```xml
<properties>
  <reachability.version>1.0.0-RC5</reachability.version>
</properties>

<dependency>
  <groupId>us.hebi.graalvm</groupId>
  <artifactId>reachability-annotations</artifactId>
  <version>${reachability.version}</version>
  <scope>provided</scope>
</dependency>

<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <version>3.13.0</version>
  <configuration>
    <annotationProcessorPaths>
      <path>
        <groupId>us.hebi.graalvm</groupId>
        <artifactId>reachability-processor</artifactId>
        <version>${reachability.version}</version>
      </path>
    </annotationProcessorPaths>
    <compilerArgs>
      <arg>-Aproject=${project.groupId}/${project.artifactId}</arg>
      <arg>-Areachability.outputFormat=1.0.0</arg>
    </compilerArgs>
  </configuration>
</plugin>
```

- Since JDK 23 `javac` does not run annotation processors found only on the classpath: `annotationProcessorPaths` (above) is required, or `<proc>full</proc>`.
- Set `outputFormat` explicitly. `1.0.0` (default) writes `reflect-config.json` and `resource-config.json` with `typeReachable` conditions. `1.2.0` is the combined `reachability-metadata.json` with `typeReached` (needs the class to be INITIALIZED, which can break a lookup that runs before initialization) and supports globs. Delete `target/classes/META-INF/native-image/reachability-generated` before switching format: old files are not removed and get packaged with the new ones.
- Generated output: `META-INF/native-image/reachability-generated/<project>/`, picked up from the classpath by native-image with no further configuration.

## Worked recipe 1: class, resource, bundle looked up by computed name

A trial program does three lookups whose names are computed at run time (so nothing folds): `Class.forName("pkg.plugins." + name).getDeclaredConstructor().newInstance()`, `getResourceAsStream("/templates/" + name + ".txt")`, `ResourceBundle.getBundle("pkg." + name)`. Built with no annotation and `--exact-reachability-metadata`, each run dies with the entry to add:

```
MissingReflectionRegistrationError: Cannot reflectively access the class 'trial.reflect.plugins.Hello'. ... { "type": "trial.reflect.plugins.Hello" }
MissingResourceRegistrationError: Cannot access resource from module 'null' at path 'templates/hello.txt'. ... { "glob": "templates/hello.txt" }
MissingResourceRegistrationError: Cannot access resource bundle with name 'trial.reflect.messages'. ... { "bundle": "trial.reflect.messages" }
```

The fix is one annotation on the class that does the lookups (the plugin classes themselves are not referenced anywhere else, so without registration they are not even in the image):

```java
@Reachable(
    classes = {Hello.class, Bye.class},
    memberAccess = ALL_DECLARED_CONSTRUCTORS,          // static import of MemberAccess
    resources = "/templates/*.txt",                    // leading / = classpath root
    bundles = "messages")                              // relative to the annotated class's package
public class Main { ... }
```

Imports: `us.hebi.graalvm.reachability.annotations.Reachable` and `MemberAccess`. Generated (1.0.0 format):

```json
[{ "condition": { "typeReachable": "trial.reflect.Main" }, "name": "trial.reflect.plugins.Hello", "allDeclaredConstructors": true },
 { "condition": { "typeReachable": "trial.reflect.Main" }, "name": "trial.reflect.plugins.Bye",   "allDeclaredConstructors": true }]
```
```json
{ "resources": { "includes": [{ "condition": { "typeReachable": "trial.reflect.Main" }, "pattern": "templates/[^/]*\\.txt" }] },
  "bundles": [{ "condition": { "typeReachable": "trial.reflect.Main" }, "name": "trial.reflect.messages" }] }
```

After rebuilding, all four runs (both plugins, the resource, the bundle) succeed; the image is about 9.7 MiB and builds in about a minute with `-Ob`.

Choosing the annotation (from `BEST_PRACTICES.md`; put it on the class that DOES the lookup, not on the target, unless Java code already reaches the target):

- class with native methods that look up its own fields: `@Reachable(jniAccessible = true)` on that class;
- classes a native library finds with `FindClass`: `@Reachable(jniAccessible = true, classes = {...})` on the Java class that loads the library (a condition on a class that no Java code ever references never fires);
- one method or field looked up by name: `@ReachableMember`;
- a class instantiated from a string (a skin, a plugin): `@Reachable(condition = Control.class, memberAccess = ALL_PUBLIC_CONSTRUCTORS)` on that class;
- third-party libraries you do not control: one holder class with one repeatable `@Reachable` per library, conditioned on a type that shows the library is used;
- JavaFX views: `@ReachableFxView` or `@ReachableFxResources` (next recipe).

Specifying any of `classes`, `classNames`, `resources`, `bundles`, `proxies` means the annotated class itself is no longer registered; add it to `classes` if it needs metadata too. `condition = Object.class` makes an entry unconditional.

## Worked recipe 2: a JavaFX FXML view (with StaticFX, see `staticfx.md`)

`@ReachableFxView` on the class that loads the FXML parses the view's `.fxml`, `.css` and localized `.properties` files and generates the metadata for the controller, its `@FXML` members and the JavaFX node types the FXML names. Naming convention: `GreetingView.java` pairs with `greeting.fxml`, `greeting.css`, `greeting*.properties` (lower-cased view name, same package folder in resources).

```java
@ReachableFxView
public class GreetingView {
    public static Parent load() throws IOException {
        return FXMLLoader.load(GreetingView.class.getResource("greeting.fxml"));
    }
}
```

What the processor wrote for a view with a `VBox`, a `Label`, a `Button`, a controller and one stylesheet: reflection entries for the controller (`trial.hellofx.GreetingController`) and `javafx.scene.layout.VBox`, `Pane`, `Region`, `javafx.scene.control.Label`, `Labeled`, `Button`, `ButtonBase`, `Control`, `Parent`, `Node`, `javafx.geometry.Pos`, all conditioned on the view class, and resource entries `\Qtrial/hellofx/greeting.fxml\E`, `...greeting.css\E`, **`...greeting.bss\E`** and `trial/hellofx/greeting[^/]*\.properties`.

- The `.bss` entry matters: JavaFX probes for a pre-compiled `.bss` next to every stylesheet, and in exact mode that probe is an error even though the file does not exist. `@ReachableFxView` and `@ReachableFxResources` add the `.bss` names and the `@2x` to `@4x` image names; registering stylesheets through `@Reachable(resources = ...)` does not. Register stylesheets and images used from Java code through the Fx annotations too.
- **Use explicit FXML imports** (`<?import javafx.scene.layout.VBox?>`). With wildcard imports `FXMLLoader` probes `javafx.scene.control.VBox` before `javafx.scene.layout.VBox`, the probe is an error in exact mode, and the processor does not register it (upstream issue: https://github.com/HebiRobotics/reachability-annotations/issues/7; the workaround and the registered-probe alternative are in `traps.md`).
- To COMPILE a controller with `@FXML` you need `org.openjfx:javafx-fxml` on the classpath: `javafx-controls` does not bring it (`package javafx.fxml does not exist`).
- Several views: `@ReachableFxResources({"/views/*.fxml", "styles/*.css"})` (patterns starting with `/` are relative to the classpath root, others to the annotated class).

## What the annotations do NOT cover: FFM downcalls

The processor has no support for the Foreign Function and Memory API. A program that calls `Linker.nativeLinker().downcallHandle(...)` builds fine and runs until the first downcall, then dies:

```
MissingForeignRegistrationError: Cannot perform downcall with leaf type (long,long)int. To allow this operation, add the following to the 'foreign' section of 'reachability-metadata.json' and rebuild the native image:
  "downcalls": [ { "returnType": "jint", "parameterTypes": [ "void*" ] } ]
```

Write the file by hand, one descriptor per distinct function signature (`jint f(void*)` and `jint f()` are two entries; the error names only the first that failed, so register all signatures the code uses), at `src/main/resources/META-INF/native-image/<groupId>/<artifactId>/reachability-metadata.json`:

```json
{
  "foreign": {
    "downcalls": [
      { "returnType": "jint", "parameterTypes": ["void*"] },
      { "returnType": "jint", "parameterTypes": [] }
    ]
  }
}
```

Upcalls use `upcalls` (and `directUpcalls` with `class` and `method`). The trial also passed `--enable-native-access=ALL-UNNAMED` as a build arg (accepted; whether the image works without it was not tested). With these, a small CLI that deletes Windows reserved-name files through `DeleteFileW` and `GetLastError` works in the native image; the same code without the file failed on the first delete.

## Traps seen

- The generated directory is not cleaned when you change the output format (above).
- The community metadata repository (`metadataRepository`) is a different thing: it ships ready-made metadata for popular third-party libraries and is used by the Maven plugin automatically unless disabled. The trials disable it to prove their own metadata; a project with many third-party reflective libraries should leave it enabled.
- A lookup that is MEANT to fail (an optional class probed with `Class.forName`, a probed file) is still reported by exact mode, and `catch (ClassNotFoundException)` does not catch the registration error. Register such probes by exact name.
- Bundle names are compared exactly as passed: `trial.reflect.messages` does not cover `getBundle("trial/reflect/messages")`.
- Resource URLs are not percent-encoded in a native image: avoid spaces in resource names.
