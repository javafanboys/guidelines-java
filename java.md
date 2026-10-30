# Java guidelines — the master file

Read this before writing even one line of Java. It holds the rules that apply to every Java project, and below them the list of specialised files with when to read each.

**Paths.** Every repository is assumed to sit at `<codedir>/<org>/<repo>`, mirroring GitHub, where `<codedir>` is the folder you clone into. Paths in these files are relative to the file they are written in, so `../../luvml/luvml/luvml_tutorial.md` from here is the `luvml` org's `luvml` repo. If a referenced repo is not cloned, clone it from `https://github.com/<org>/<repo>` first.

## Read these when the situation arises

- **Building a JavaFX UI** → `javafx.md` (JavaFX conventions, icons, tray, keyboard-first navigation).
- **Building anything that stays running** (tray app, daemon, gate, resident server) → `resident-apps-memory.md` (idle trim, GC and memory flags). Every jr app also needs the GC section there, because every jr app names its collector.
- **Deploying to Maven Central** → `maven-central-publishing.md` (Maven 3 for deploy, the silent no-publish failure, limits, snapshots, parent POMs).
- **The user shows inclination for self-typed generic interfaces** → `self-typed-generics.md`.
- **A one-off script or small coding task** → `small-java-scripts.md`.
- **Using ArcadeDB through datapotter-arcadedbhelper** → `arcadedb.md`, the complete worked guide (graph modelling, links, embedded types, the database-service pattern, CRUD, the type-safe query API, transactions, lifecycle; earlier versions are in git history). Also `arcadedb-datapotter-migration.md` (old xyz-jphil coordinates to datapotter) and `arcadedb-logging.md` (ArcadeDB's java.util.logging into SLF4J/Logback).
- **Calling native code through FFM with jextract** → `jextract.md`.
- **Measuring Java code performance** (benchmarks, profilers) → `performance-measurement.md`.
- **GraalVM native image / AOT** (Oracle GraalVM 25.4, reachability annotations, StaticFX for JavaFX) → `graalvm.md` and the notes in `graalvm/`. Read them first; whoever learns something new about GraalVM writes it back there, so it is never re-learnt.
- **TeaVM (compiling Java to native C, or to JS/Wasm)** - if using TeaVM for ANY purpose, DO NOT proceed (choosing a C compiler, writing @Import bindings, debugging a compile/link/runtime error) without reading `teavm.md` first. It carries hard-won toolchain gotchas (which C compiler actually pairs with TeaVM's C backend, required shims, compile flags, classlib traps like System.exit being a no-op) discovered in littlejlib/jr PRP-07 - rediscovering these from scratch each time wastes real effort.
- **Any C code, or Java/TeaVM code touching raw pointers** → `../guidelines-coding/c-memory-safety.md`.

## MAVEN PROJECT CONFIGURATION, GIT PROJECT CONFIGURATION, AND FOLDER NOMENCLATURE DEFAULTS

- While creating a new piece of java code, if projectId, groupId and packagename is not explicitly specified by the user follow these defaults.
- **ORGANIZATION NAME** : let us say the organization name is `<organization-name>` ; then for this the github organization will be `https://github.com/<organization-name>` and naturally because of this the maven group id will be `io.github.<organization-name>` (we always asssume we don't have a dedicated `.com` domain for any of our organization).
- **MAVEN PROJECT/MODULE NAME AND JAVA PACKAGE NAME** : however our java packages will not be named in such a complex way, our packages will always start with the `<organization-name>` to avoid our folder heirarchy becomes too deep and non-intuitive and overly verbose without meaning or value addition; So if organization name is `datapotter` the first package is `datapotter`. If the (sub-)organization name is `datapotter-ai` the package will be `datapotter.ai` (as this is a sub-organization), and groupid: `io.github.datapotter` (avoid sub-group id : `io.github.datapotter.ai` as it is complex). If the organization name is `xyz-jphil` the groupid is : `io.github.xyz-jphil` (because jphil is not a sub-organization to xyz, it is just the organization name itself)
	- project name will have the organization name as a prefix, this makes the navigation amongst 100s of project simpler and predictable in netbeans or anywhere they are listed one after the the other. So let us say take an take of orgname `datapotter` and project is about a document management system and let us say it is AI driven. Then the artifact id will be `datapotter-ai-document_management_system` , groupid: `io.github.datapotter` , java package where all code implementatoin etc. would be `datapotter.ai.document_management_system`; literally if you notice the artifact and package are mirroring each other just the dashes are replaced with a dot. This pattern can be thought of artifact name as `organization-project_category-project_name` and for package `organization.project_category.project_name` . This is the most common method we are following here.
		- an exception to package name is, for example if the organization name is `prosemirrorfx` we might want to use the shortname for the project `pmfx` this is a deviation and allowed.
	- however the project folder itself will be named just `document_management_system` so you have `<codedir>\datapotter-ai\document_management_system`
	- if you have even deeper heirarhy, example artifact id `datapotter-ai-monitoring-db` the folder structure will be `datapotter-ai\monitoring\db` (notice is it not `datapotter\ai\monitoring\db` which means the depth is controlled to match the depth we have in github. In a github organization all repositories are flat, no nesting, so we also follow the same system, if we want nesting, we make a new organization itself with the convention `${parentorganization}-${subgroup}` in this example parentorganization=datapotter and subgroup=ai.
		- Alternatively if you want to have one more level of nesting, the alternative is to have an entire group of projects in a same github repository (not one repository per project). DataHelper is an example where a bunch of projects were made into a single repository. Both conventions are fine, and it is a user decision.
		- By default we prefer one project one repository, unless the projects are very strongly coupled, this is the real criteria for them to have a common repository, the depth of heirarchial organization is not the criteria.
- **PROJECT HOME FOLDER CONVENTIONS** : For example the name of the organization is `littlejlib` and the name of the project is `netrec` , then the folder where you must save user configuration settings/files is `~/littlejlib/netrec` << THIS IS CORRECT.
- DO NOT MAKE A FOLDER `.netrec` << this is typical convention, but in this convention we are not able to group projects per organization where they share settings etc. Secondly, even the system of naming `.<orgname>` is deferred by us, yes it makes the folder hidden in linux, but we are simply not interested. We have a consistent uniform naming system accross, maven artifact id, package name, github project name, git repository name, settings folder name, etc; they all follow a same consistent pattern.
- **MVND** : DO NOT prefer using `mvnd.exe` instead use `mvn.cmd`, because through `mvnd.exe` is faster it is unstable and unreliable. So if `mvnd.exe` gives problem please simple use `mvn.cmd` .
- **Maven Dependencies Version** : Can you please search the internet and validate assumptions regarding which maven dependencies exist and which version should we use, otherwise i found that you always end up using wrong version and then only during compliation time discover get it. Consider taking help from mvnrepository.com ,  central.sonatype.com, search.maven.org,  repo1.maven.org etc.
- **Maven testing command** :	To run special test (with no `@Test` that is which are not junit tests), or trials, or experiments files in the `Test Packages` in maven in the `src\test\java` which just have a main method, you can use ```mvn exec:java -Dexec.mainClass="main.class.fullname" -Dexec.classpathScope=test```
- **Main class name** - Main class name should be saved in a project/artifact property variable, this should be such that it works with (i) maven exec, (ii) it works with uber-jar / shade, (iii) it works with javafx plugin (iv) it works with build with dependencies plugin (if used), (v) it works with graalvm, think wherever main is needed you should use this variable. If javafx plugin is used, mostly maven exec is not relevant, either one only will be used. However just be aware that we don't want duplication and mismatch of the value across these 3.
- **Maven shade** : In the maven shade, you might need to filter and exclude META-INF/*.SF, META-INF/*.DSA, META-INF/*.RSA , and other manifest files. this is common bug/issue which arises when using shade.

## JAVA CODING STYLE PREFERENCES

- **JDK25** : Use all features upto jdk25, including FFM. Minimum use jdk21.
- **TERSE-CODE** : DO NOT Write verbose code (which makes non-Java coders dislike even looking at java source code), make the code appear terse, elegant and easy. Most of all rules and guidelines mentioned below follow the terse-code preferance principle.
- **Promote common patterns frequently used in the codebase into utility static functions etc to reduce boiler plate** .
- **VAR** : Strictly use var for all local variable initialization.
	- The reason are:
		- uniformity and readability
		- var usage makes syntax shorter
		- syntax becomes closer to javascript making the code look more similar to js and thus more approachable to non-java programmers.
		- Additionally by using `var` we can reduce number of import statements required. Example - `var first = "hello".chars();` here importing IntStream was not required because we used var. And further invocation along this chain yield more type likes list etc., we may not need import them also, making overall syntax shorter.
	- Usage Example:
		- For non-null initializations of local variable : `var count = 0;` `var name = "test";` `var list = new ArrayList<>();`
		- For null initializations: `var name = (String)null;` OR `String name = null;` both are ok, however either way NEVER use explicit type declarations like: `String name = "test";`
- **PACKAGE-PRIVATE FIELD VARIABLES** : Avoid making class field variables as private, leave it as package-private, it is terser and more flexible and sufficiently safe. Example : `public class SomeDto { Object x; int y; }` notice we didn't prefix `private`.
- FUNCTIONAL SCALA LIKE JAVA : Write Java code applying Scala style coding patterns and functional programming patterns to make code more compact and less verbose. Scala coding patterns could be a reference, but write java code. Example use switch over sealed types.
- Refer new features from project amber.
- User switch expression over pattern matching.
- Refer to luvml, luvs, luvdocx as a flagship pattern demonstrating these patterns.
- **DO NOT USE MULTILINE STRING TEMPLATES** `"""` pattern, as it has been removed from java and again pushed back to the drawing board. Instead, store such multi line strings/string-templates in it's own file, such as in the maven project resources folder or it's subfolder. Pull the data out and replace the template variables if needed. For this write a static utility function once (or reuse if already existing) and reuse throughout wherever needed.
- **USE LUVML - FOR HTML STRING TEMPLATES** - use luvml, luvs, etc. `../../luvml/luvml/luvml_tutorial.md` ; never use stringbuilders.
	- **CSS goes in luvs, not in `.css` files or Java strings**, with class names as `CssClass` enums and colours as a `CssVariable` enum (tutorial `../../luvml/luvs/luvs_tutorial.md`). A domain enum can itself be the class (`enum Status implements CssClass`). luvs needs the code in a NAMED package: Java cannot static-import from the default package, so a flat jbang folder gets a package and a one-line default-package launcher.
	- **Before writing luvml/luvs, read `../../luvml/luvml/ai-expectations.md`; whenever the API surprises you, append an entry there** (expected / happened / did / FIX-or-TEACH proposal). The user's aim (2026-10-30) is an API an AI can write by pure guess; the file is how the surprises get fixed or taught instead of rediscovered every session. It is public in the luvml repo, so write entries in neutral terms (no project, people or organisation names).
- **DO NOT USE TOO MANY INNER CLASSES** : Do not make inner java classes, either use lambda class declaration if possible or dedicated files for each class, do not make inner classes.
	- AI defaults tend to create excessive inner classes which are supposed to be public to be able to be accessible from the overall code. So please prefer NOT having inner classes, make them as normal public classes.
	- So all inner classes must be made into public classes in their dedicated files, even the small inner classes. This prevents per java file from becomming huge and and easier to maintain (this goes with our Keep all source files (.java) under 200 lines rule)
- **SMALL JAVA FILES** - Do not make classes which are very BIG (many lines), break them down into smaller files, grouping functions and features thematically / logically to increase readability and ease of maintainance.
- **STATIC IMPORTS** : Utilize static imports for utility functions, and constants. **Benefits** :
	- Eliminates repetitive class name prefixes
	- Makes code more readable and closer to functional languages
	- Particularly effective for enums, constants, and utility methods
	- Reduces visual noise and improves code density
	- Should be used by default when referencing enum values or constants multiple times
- **STRICTLY START USING IMPORT MODULE FEATURE OF JDK25**
	- `import module javafx.base;`
	- This is a new feature OLD code will contradict, but this the new enforced standard don't refer to old code and get tricked
- **MAKE IMPORTS MORE CONCISE USING WILDCARDS AND MODULE IMPORT JDK25 FEATURE**:
	- VERBOSE DO NOT DO ❌
	```java
	// ❌ Verbose: Multiple individual imports
	import javafx.scene.Node;
	import javafx.scene.control.Label;
	import javafx.scene.control.ProgressBar;
	import javafx.scene.layout.HBox;
	import javafx.scene.layout.Priority;
	```
	- CONCISE ✅ GOOD :
	```java
	// ✅ Concise: Wildcard imports
	import javafx.scene.*;
	import javafx.scene.control.*;
	import javafx.scene.layout.*;
	```
	- TERSEST ✅ ✅ 🚀:
	```java
	// 🚀 Tersest: Single module import (JDK 25+)
	import module javafx.controls;
	```
	- You might face conflicts after doing such imports, handle it as follows:
	```java
	// explicit single-type import wins over module import
	import java.sql.Date; // resolves ambiguity — takes precedence over any module import
	```
	or use a fully qualified name inline:
	```java
	var date = new java.util.Date(); // notice the use of var
	```
	- When you use wild card in imports, you might end up with conflict, that also can be gracefully handles by var, and sometimes using class full name (with full package name) in certain context, because the benefit of wild card import is much more sometimes that these one odd conflicts should not stop us from using simpler and more compact import syntax which makes our overall code shorter.
- **STRICTLY FOLLOW THE TERSE STATIC DECLARATION PATTERN FOR SIMILAR TYPES**
	- Instead of:
	```java
	public final class Fonts {
		public static final String DEFAULT = "Arial";
		public static final String TAMIL = "Latha";
		public static final String DEVANAGARI = "Mangal";

		private Fonts() {}
	}
	```
	- Declare like this:
	```java
	public final class Fonts {
		public static final String
			DEFAULT = "Arial",
			TAMIL = "Latha",
			DEVANAGARI = "Mangal";

		private Fonts() {}
	}
	```
- **POJO & DTOs - plain old java objects - data transfer objects**
	- For any POJO class (any class whose purpose is to hold data), always use @Data from either DataHelper or Lombok. **NEVER WRITE MANUAL GETTERS/SETTERS** (either use Lombok or DataHelper). If not clear which to use prefer DataHelper and its `@Data` annotation over Lombok, otherwise the logic to decide when to use what refer - `../../datapotter/datahelper/README.md`
	- **DATAHELPER IS NOT AN ARCADEDB THING.** Do not let the ArcadeDB bullet further down be where you first meet it, and do not conclude it is only relevant when a database is involved. `datapotter.datahelper.Data` is a standalone DTO generator: fluent + bean accessors, generated `equals`/`hashCode`/`toString`, type-safe `$field` symbols, reflection-free by-name property access (so it survives GraalVM native and TeaVM), and a generated immutable record projection `Xxx_R` with `toRecord()` / `toMutable()`. `@ArcadeData` (module `datapotter-arcadedbhelper`) is merely one optional persistence trait layered on top of it.
	- **THE RULE, AND IT NEEDS NO THINKING: any class whose job is to hold data gets `@Data`. Write `@Data`, do not deliberate, move on.** Never weigh component counts, never audit call sites, never ask yourself whether it "feels immutable" — those are things you cannot know before the code exists, which is exactly why the earlier version of this rule failed.
		- **A `record` is permitted ONLY when it matches one of these four. The list is closed.**
			- (1) It is declared INSIDE another class or method as a throwaway tuple and never leaves it (a local pair, a private grouping for one loop, a nested `record Footnote(...)` used only by its enclosing class).
			- (2) It is used as a key in a `Map`/`Set`, or is compared whole with `equals` as a matter of design.
			- (3) It is a case of a `sealed` hierarchy that gets pattern-matched.
			- (4) It enforces an invariant in a compact constructor and carries no settable state — a validated value object, not a carrier of fields.
		- **No match, or any doubt about whether one matches, means `@Data`.** Doubt resolves to `@Data`, always. Being wrong towards `@Data` costs nothing; being wrong towards `record` has already cost a rewrite.
		- **A record only ever makes sense under 100% certainty that the type will never be modified or copied. You will almost never have that certainty in advance, so in practice: `@Data`.**
		- **Tripwire — the moment you write a `withX(...)` copy method on a record, you chose wrong; it must be `@Data`.** A wither that re-lists the components IS a builder, and DataHelper's fluent mutable class plus `toRecord()` is that builder with nothing to hand-maintain. Do not "fix" it by adding more withers.
		- **Choosing `@Data` gives up NO immutability.** It generates the immutable record projection `Xxx_R` (`mutable.toRecord()`, deep through nested DTOs/Lists/Maps), so you get the safely-hashable snapshot on demand — plus the `$field` symbols, reflection-free by-name access, generated `equals`/`hashCode`/`toString` and the serialization traits, all of which a bare record gives up. There is no immutability argument for a record. There never was.
		- **BEFORE reasoning about what `@Data` can or cannot do, READ THE GENERATED CODE** in `target/generated-sources/annotations/<pkg>/Xxx_A.java` / `_I` / `_IR` / `_R`. It is plain readable java and it is right there. Do not answer from the README's summary, and do not invent a workaround for a limitation you have not confirmed exists — that mistake has already produced a needlessly complex design.
		- When converting an existing record to `@Data`, **do not go looking for a new home for whatever the compact constructor was doing** — the same design is available three ways: a plain field initialiser (`List<SourceRef> sources = List.of();`), or your own constructor on the final class (`@Data` does not prevent you declaring one), or an override of the generated getter. On the override route the generated code answers how many you need: the fluent getter `x()` is a `default` in `_IR` that delegates to `getX()`, and `getPropertyByName()` and `toRecord()` route through `getX()` as well — so overriding the BEAN getter alone propagates to all of them, and the fluent accessor does not need touching.
		- Two things do still need checking rather than assuming when converting: the emitted JSON must be compared against real stored files before and after, because the on-disk shape is a contract; and Jackson annotations go on the FIELD (see the Jackson interop section of the DataHelper README).
	- If at all using lombok STRICTLY prefer Fluent Lombok and if using lombok ALWAYS USE version 1.18.38 or higher)
		- Use @Data or @Getter & @Setter etc. And please prefer fluent getters and setter, use `@Accessors(fluent = true)` ; and @NoArgsConstructor is also ok to use
		- Reduce boilerplate, strongly prefer fluent getters and setters rather than verbose getter and setters, for both pojos and interfaces. For mutable pojos always use lombok version 1.18.38 or higher with fluent accessor annotation, for immutable pojos java records should be preferred over hand-rolling immutability in Lombok (`@Value` / `@Builder` / `@With`). **THIS SENTENCE IS SCOPED TO LOMBOK AND IS NOT THE GENERAL RULE** — it is not a licence to prefer a bare record over DataHelper's `@Data`; for that decision use the `record` vs `@Data` rule above, which is where this one has previously been misread.
		- When using Lombok and Jackson together don't use @JsonPropertyOrder and also have @JsonProperty above the fields, it IS needed
- **Self-Typed Generic Interfaces Pattern** - if user shows inclination for this pattern refer `self-typed-generics.md` and luvml (if already part of the project - see `../../luvml/luvml/luvml_tutorial.md` , read only when needed)
- **Commandline cli picocli** - Use picocli for command line input wherever applicable / needed. For naming the command line flags as constants (symbols), use DataHelper `@Data` and its generated `$field` symbols; the older `@GenerateSymbols` annotation is retired and must not be used.
- **Command line colored JLine** - If user wants richer cli interface, use JLine for making a nice command line interface with colors, and good looking at par with modern tools made using node.js, python etc. Don't use tools which make the command line incompatible with windows.
- **Keeping Source code Small and breaking projects into resuable smaller modules so that is easy to maintain**
	- Keep all source files (.java) under 200 lines
	- Proactively split files that approach this limit without waiting for instruction
	- Extract reusable components into separate classes
	- Keep all methods under 30-50 lines (viewable in one screen)
	- Break complex functions into smaller private helper methods
	- Each function should have a single responsibility
	- Big code needs to be broken into smaller files, and large packages need to broken into multiple maven projects. If not done, it will be difficult to maintain
- **Comments** - I don't know your defaults regarding comments, but I prefer NO comments in the source code, unless really really needed. The reason is comments become obsolete as the code updates and we don't update comments. So the single source of truth is source code. Use comments very rarely, and when really needed.

## Logging preferance

- I want terse logging.
- In case of CLI programs where the user sees the console, the logs should be in stderr or file, you need to discuss with the user and have clear channels for logs and actual output user is interested in. The logging mentioned here, can be outputted as is in terminal only for server kind of apps, for CLI you need to plan better. For GUI apps also u can treat it like server appis. When it is a mix of gui and cli ... again you should plan. I think WARN and higher errors you can output in console even for CLI project, but against decide with user.

## Database library preferance

- Read this first: `datapotter-arcadedbhelper` / `@ArcadeData` below is DataHelper PLUS one optional persistence trait. DataHelper's own `@Data` stands alone, has nothing to do with any database, and is the default for DTOs everywhere — see the POJO & DTOs section above. Do not infer from this bullet that DataHelper is an ArcadeDB feature; that inference has already caused a wrong POJO decision.
- When using local database prefer Arcadedb (local).
	- **2026-09-11: moved to a new GitHub org/groupId** — `xyz-jphil/datahelper` split into `datapotter/datahelper` + `datapotter/arcadedbhelper`, groupId `io.github.xyz-jphil` -> `io.github.datapotter`, package `xyz.jphil.arcadedb.datahelper` -> `datapotter.arcadedbhelper`.
	- Current baseline: **ArcadeDB 26.7.3 + datapotter-arcadedbhelper 1.7** (annotation processor is now 1.3 too, up from 1.0). The API is typed on both sides: `Upsert` for field-level writes, `Query` for lazy typed reads (`query(db, X.TYPEDEF).eq($f, v)` — declare `.factory(X::new)` once in the model), `Traverse` for adjacency when you hold the `Vertex`. Do not use datahelper 1.0 with ArcadeDB 26.x — it compiles but throws `NoSuchMethodError` at runtime, silently killing every `Link<T>` resolve/write and all `$out`/`$in`/`$both` traversal. Also pin **Gson 2.14.0**: 26.x calls `com.google.gson.Strictness` which does not exist before Gson 2.11, so an older Gson anywhere in the tree kills ArcadeDB at startup while the build stays green.
- **Local DB default = ArcadeDB, NOT SQLite — and AI must override its SQLite reflex.**
	- LLMs are trained to reach for SQLite as the "safe/boring" local DB. For our projects that default is usually wrong. ArcadeDb code will be **terser** - 40% lesser than SQLite+JDBC for the same CRUD, it has compile-time safety, it is reflection-free. Arcadedb is multi-model document (nosql db) + sql db so support **Fluid/evolving schema** with *queryable* nested documents. It has a server and inprocess embedded mode. It is a bit slow to boot compared to sqlite, but with a resident server and/or our `jr.exe` AOT cache it is good. I don't see any reason to use or sqlite when we have arcadedb.

## UI - javafx vs swing

- I clearly prefer javafx by far. Swing is a mistake to use.
- COMPULSORILY USE THIS AS `../../luvml/luvjfx/README.md` FOR MAKING JAVAFX UI.
	- You should also use user made javafx .fxml (but u don't need to make them, u can just help in maintaining them)
- Before building any JavaFX UI, read `javafx.md` in this folder.

## Special tools and libraries

- For json or toml reading/writing you can use jackson. Jackson works nicely with datahelper that is another good integration we have. `jackson-dataformat-toml` for toml is fine.
- For Html prefer jsoup, and even for xml prefer jsoup (yes defer jaxb it is fragile)
- **Tersejson and terserest** - whenever handling json creation for the purpose one time rest calls ... and we wish to do things very very tersely without creating and DTOs ... please see this for approaching rest apis from java. `../../littlejlib/tersejson/README.md` and `../../littlejlib/terserest/README.md` (coordinates `io.github.littlejlib:littlejlib-tersejson` / `littlejlib-terserest`, package still `terse`; the old `io.github.xyz-jphil:tersejson` / `terserest` are superseded)
- For making servers prefer javalin. Javalin goes nicely with luvml.
- jsoup for html and even xml parsing.
- **Java runner launchers** - `jr.exe` (github.com/jarrunner/jr) can be copied to something like xyzapp.exe and placed in a folder on the PATH, then with it's companion xyzapp.jrc (locarted as a sibling file), we can invoke any java jar file (or even java class file). It also handles graceful and automatic AOT cache creation, cleanup and re-creation when it comes stale. This AOT feature can be disabled. This give a nice `.exe` for running java with (optional) AOT which makes it faster. AOT is optional but by default in enabled. For a .jrc file sample run `jr.exe -Xjr:create-config`. All our command line tools are usually made accessible as exe with jr.exe , we prefer this over .bat file. jr does one more thing, if you open it outside of cmd.exe console window is suppressed it behaves like how javaw.exe would, but from console it runs how java.exe would. All this is provided out of box.
	- **The TeaVM build (PRP-24/26), all as .jrc keys:** `java.type=jre|jdk` (auto-install now fetches a JRE by default); `run.url=<https url>` or `run.maven=group:artifact:version`, plus a required `run.sha256=`, to fetch the jar itself on first run. The jar goes to `~/.m2` for run.maven; downloads resume and show progress; the jar must be shaded, since dependencies are not fetched. The .jrc can also be embedded inside the exe (`-Xjr:edit=app.exe -Xjr:resource.RCDATA.JRC=app.jrc`), so one exe needs no companion file. Branding: `-Xjr:edit=app.exe -Xjr:icon=x.ico -Xjr:version=...`.
	- **Icons:** stamp a 256x256, 8-bit, PNG-compressed entry (~5-15 KB); Windows scales it down well for 32 px and up. At 16-20 px (a caption icon) a scaled-down design smears and no resampling fixes it, so add small sizes DRAWN as pixel art (~500-800 bytes each) when the small icon visibly suffers (jr does since 2026-10-03, PRP-31). ImageMagick's `-define icon:auto-resize` writes uncompressed BMP entries and 16-bit PNGs, which bloated jr's icon to 110 KB.
- **`jacli` (jack-lee, github.com/littlejlib/jacli) — the compiler-accurate answer for any question about Java SYMBOLS** (declarations, callers, implementations, a dependency's API straight from `~/.m2`). Run `jacli claude install` once: it writes jacli's own agent prompt and hooks into Claude Code, so that installed prompt is the one copy of how to use it; `jacli claude status` says whether it is current.
	- Without it installed, `jacli` alone prints the short help and `jacli tutorial` the full one.
- **unicode support in terminal in windows enviroment** : in java + windows terminal combination, before starting the java app we need to run `chcp 65001` (or `chcp 65001 > nul` to silence it) to make sure unicode renders properly — some client apps handle this internally already, but most java apps don't and will break/mojibake if this isn't done first. other alternative is to use this library `../../littlejlib/windows_console_set_unicode_output/README.md` (coordinates since 2026-10-30: `io.github.littlejlib:littlejlib-windows_console_set_unicode_output:1.0`, package `littlejlib.windows_console_set_unicode_output`; it used to be `io.github.xyz-jphil:xyz-jphil-windows_console_set_unicode_output`, package `xyz.jphil.windows_console_set_unicode_output`, and a project still on the old name keeps building from `~/.m2` until it is moved; new code uses the new name)

## IMPORTANT INSTRUCTIONS RELATED TO JAVA CODE EDITS AND INSERTIONS

- When claude code is writing new code, it is generally very precise and good, but when it is doing edits (especially after reading raw dumps from javap instead of correctly using jacli or reading the source code etc) it start writing full java class names, instead of short names and adding imports (adding specific imports, or wildcard imports or module imports)
- Example of a bad new code insertion :
	```
	+ java.lang.String newFunctionAdded(java.land.Integer inputParam1, someuserpackage.somesubpackage.SomeClass1 someclassobj)
	+ etc.
	```
- this should have been more like :
	- at top:
		```
		+ import someuserpackage.somesubpackage.SomeClass1;
		```
	- in code area:
		```
		+ String newFunctionAdded(Integer inputParam1, SomeClass1 someclassobj);
		```
- if you mess this up, fine, but somewhere we need to correct this, at some later pass, a lot of code here and there in various projects has been polluted like this, it spoils readability and sets the strong trend for the AI to follow for later cycles as well.
