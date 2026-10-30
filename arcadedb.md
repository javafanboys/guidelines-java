# ArcadeDB Working Examples with @ArcadeData (2026-09-11)

This document provides concrete, working examples using the **new @ArcadeData annotation pattern**.
Earlier versions are in this repository's git history.

> **Change since 2026-07-31 — the library moved to a new GitHub org and Maven groupId.** The old single repo was split into `datapotter/datahelper` (public) and `datapotter/arcadedbhelper` (public); the old groupId became `io.github.datapotter`, and the old package roots renamed to `datapotter.datahelper` and `datapotter.arcadedbhelper`. Coordinates and code below are updated to the new namespace, and to the current versions: `datapotter-arcadedbhelper` 1.7 (processor now 1.3 too — it moved off 1.0). Full mapping and a before/after example: datapotter-migration-guide.md (same folder). **Not yet updated in this pass:** the feature-level content below predates the enum vocab (`@AsUuid`/`@AsName`), `List<E>` enum support, embedded `@Data` block read-back fixes, and map read/write traits work (PRP-27 through PRP-30) — none of that is demonstrated here yet.
>
> **Change since 2026-06-30 — the versions moved: ArcadeDB 26.7.3, datahelper 1.3.** ArcadeDB 26.x made two source-compatible but **binary-incompatible** changes that broke datahelper 1.0 at runtime with `NoSuchMethodError`, silently disabling every `Link<T>` and `$out/$in/$both` call. Fixed in 1.1. If you have your own code holding RIDs, read [ArcadeDB 26.x: what broke and what to change](#arcadedb-26x-what-broke-and-what-to-change) before upgrading.
>
> **Also new — the API is now typed on both sides.** 1.2 added [`Upsert`](#8-typed-field-level-upsert-upsert) (typed field-level writes) and made the instance DSL work on `@ArcadeData(type = VERTEX)` classes. 1.3 added [`Query`](#6-typed-reads-query) (typed, lazy reads — the last place string literals were unavoidable) and [`Traverse`](#7-vertex-addressed-adjacency-traverse) (adjacency for a caller holding the `Vertex`), plus `TypeDef.factory(X::new)` so a read reads `query(db, X.TYPEDEF)`.

> **Change since 2026-02-12 (PRP-04):** the manual `addLinkType` / `$$`-prefix LINK approach is **retired**. References are now first-class **transient reference types** — `Link<T>` / `LinkList<T>` / `LinkMap<K,T>` — with reflection-free resolution and graph traversal. See [Transient Reference Types](#prp-04-transient-reference-types-link--graph-traversal) below.

> **Query guidance (2026-07-01):** For reads, **default to the native `db.select()` DSL — not `db.query("sql", ...)`.** The DSL handles compound `AND`/`OR`, ranges, `in`, null checks, ordering, paging, `count`/`exists`, and graph traversal; SQL strings are for a short list of genuine exceptions only. The full rationale, complete operator reference, and worked examples are in [§2 Query API Options — DSL FIRST](#2-query-api-options--dsl-first) and §3–§4 of *Production Essentials*.

#### @ArcadeData - annotation 
```java
@ArcadeData              // ONE annotation, no Lombok
public final class TestPersonDTO extends TestPersonDTO_A {

    String name;         // Package-private (terser)
    String email;		 // DO NOT use private, 
    Integer age;		 // the super class needs access to this to support auto-generated getters/setters ; so use package-private which means no prefix.

    public static final TypeDef<TestPersonDTO> TYPEDEF =
        schemaBuilder()        // Pre-filled with class and fields!
            .factory(TestPersonDTO::new)
            .unique($email)
            .__();
}
```
**Benefits and intent**:
- Terser
- Type safe (we will see more of this later)
- Static language friendly - find usage, refactor rename etc.
- No two annotations, no Lombok @Data dependency
- This annotation processor is not as complex as Lombok, but quite powerful when we follow this pattern of abstract class. 

## Latest ArcadeDB Maven Coordinates (v26.7.3)
Before starting any project, please check local maven repo or internet and use the latest version 
```xml
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-engine</artifactId>
    <version>26.7.3</version>
</dependency>
<dependency>
    <groupId>com.arcadedb</groupId>
    <artifactId>arcadedb-server</artifactId>
    <version>26.7.3</version>
</dependency>
```

⚠️ **Gson floor.** ArcadeDB 26.x is built against **Gson 2.14.0** and calls `com.google.gson.Strictness`, a class that does not exist before Gson 2.11. If anything else in your dependency tree pins an older Gson (2.10.1 is common), ArcadeDB dies at startup with `ClassNotFoundException: com.google.gson.Strictness`. Pin Gson explicitly:

```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.14.0</version>
</dependency>
```

This is a *runtime* failure — Maven resolves the older Gson happily and the build is green. In a multi-module reactor that shares a parent with Google-API modules, check the effective POM (`mvn dependency:tree -Dincludes=com.google.code.gson`) rather than trusting the build.

## Helper Library Dependencies

```xml
<!-- ArcadeDB DataHelper with @ArcadeData annotation -->
<dependency>
    <groupId>io.github.datapotter</groupId>
    <artifactId>datapotter-arcadedbhelper</artifactId>
    <version>1.7</version>
</dependency>
```

**Use 1.7 (or at minimum 1.2) with ArcadeDB 26.x.** 1.0 compiles against 26.x but fails at runtime — see the next section. The annotation *processor* artifact is now also at `1.3` (it tracked the runtime library up through 1.0 for a while, but has since moved with it).

### Annotation Processor Configuration

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <source>25</source>
                <target>25</target>
                <annotationProcessorPaths>
                    <path>
                        <groupId>io.github.datapotter</groupId>
                        <artifactId>datapotter-arcadedbhelper-processor</artifactId>
                        <version>1.3</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## ArcadeDB 26.x: what broke and what to change

Two changes between 25.7.1 and 26.7.3 are **source-compatible but binary-incompatible**. Code compiled against 25.x keeps its old method references in the bytecode, so it links fine and then throws at the moment the line executes. Recompiling against 26.x is what surfaces them.

### 1. The `RID(BasicDatabase, …)` constructors were removed

26.x replaced them with static factories. The old form is gone from the jar:

```java
// 25.x — removed in 26.x, throws NoSuchMethodError at runtime if compiled against 25.x
Document doc = new RID(db, "#12:3").asDocument(true);

// 26.x
Document doc = RID.create(db, "#12:3").asDocument(true);
Vertex   v   = RID.create(db, "#12:3").asVertex(true);
```

The single-argument `new RID("#12:3")` still exists; it is the two-argument database-bound forms that went away.

**Why this one is nasty.** In datahelper 1.0 the removed constructor sat at six sites, and between them they carried the entire reference feature: vertex adjacency (`$out` / `$in` / `$both`), `Link.resolve(db)`, and the write path for every `Link` / `LinkList` / `LinkMap` field. So on 26.x the feature failed in both directions while the schema, the queries, and every scalar field kept working perfectly — which reads as "graph traversal is mysteriously broken" rather than "the library needs an upgrade". Fixed in datahelper 1.1. **If you wrote your own RID-handling helpers against 25.x, grep for `new RID(` before upgrading.**

### 2. Creating a record of a VERTEX type

`db.newDocument(typeName)` refuses a type declared `@ArcadeData(type = VERTEX)`:

> Cannot create a document of type 'X' because is not a document type

A vertex *is* a document in ArcadeDB (`VertexType extends DocumentType`, `MutableVertex extends MutableDocument`), so everything downstream of creation is identical — only the factory call differs. Dispatch on the declared schema type:

```java
var type = db.getSchema().getType(typeName);
MutableDocument rec = (type instanceof VertexType) ? db.newVertex(typeName) : db.newDocument(typeName);
```

datahelper 1.1 does this internally (`NewRecord`), so the instance DSL now covers VERTEX and DOCUMENT alike:

```java
new Person().email("a@x").name("Alice")
    .in(db).whereEq("email", "a@x").upsert();     // works on @ArcadeData(type = VERTEX) from 1.2
```

**Edges are deliberately refused** — an edge cannot exist without both endpoints, so a keyed upsert has nothing sensible to create. Build those with `sourceVertex.newEdge("Knows", target)`.

### Verifying an upgrade

Pin your test harness to the same ArcadeDB version as the library. The datahelper break went unnoticed precisely because `test-scripts/arcadedata-test` stayed on 25.7.1 while downstream projects moved to 26.x. The full suite — `LinkIntegrationTest`, `UpsertIntegrationTest`, `QueryIntegrationTest`, `ArcadeDataIntegrationTest`, `NestedTypesIntegrationTest`, `PRP21DeserializationTest`, `PRP22IntegrationTest`, `StaticFactoryTest`, `SupplierPatternTest` — is green on 26.7.3 with datahelper 1.3.

---


---

## ⚠️ MANDATORY: stop ArcadeDB writing `log/` into whatever folder you launched from

**Every project that embeds ArcadeDB must ship its own `arcadedb-log.properties` AND exclude the engine's copy from its shaded jar. Two steps, both required. This is not a tidiness matter — it has already broken things.**

### What goes wrong

`arcadedb-engine.jar` bundles its own `arcadedb-log.properties`, and the upstream default is:

```properties
handlers = java.util.logging.ConsoleHandler, java.util.logging.FileHandler
java.util.logging.FileHandler.pattern = ./log/arcadedb.log
```

`./log` is relative to the **working directory**, not to the application's home. So every process that opens an embedded database creates a `log/` folder wherever it happened to be launched from. For a CLI that the user runs from anywhere, that means a `log/` folder in whatever directory they were standing in.

`SERVER_ROOT_PATH` does **not** affect it. Setting it and expecting the logs to follow is the natural assumption and it is wrong.

### What it actually cost

- **~31 stray `log/` folders** across the code tree, full of `arcadedb.log.N` files. Several of them then collected unrelated real data because a later tool saw a `log/` folder and used it, so they cannot simply be deleted wholesale.
- Stray `log/`, `prp/`, `target/` and `test-logs/` folders in the **user home root**, from tools launched with the wrong working directory.
- The rotated names defeated the backup tool's ignore patterns: `.*\.log\.\d+` is a **full-match** regex and never matched `arcadedb.log.0.1` or `arcadedb.log.1.2`, which are the multi-segment names ArcadeDB actually rotates to. So the junk was backed up nightly.

### The fix — both halves

**1. Ship `src/main/resources/arcadedb-log.properties`** with no `FileHandler`:

```properties
handlers = java.util.logging.ConsoleHandler

.level = WARNING
com.arcadedb.level = WARNING
com.arcadedb.script.GraalPolyglotEngine.level = OFF
io.undertow.level = SEVERE
org.xnio.level = SEVERE
org.jboss.level = SEVERE

java.util.logging.ConsoleHandler.level = WARNING
java.util.logging.ConsoleHandler.formatter = java.util.logging.SimpleFormatter
```

`DefaultLogger.findConfiguredFileHandlerPattern()` looks for this as a classpath resource before falling back to the engine's copy, and with no `FileHandler` configured it never creates the directory at all.

**2. Exclude the engine's copy from the shaded jar**, so classpath ordering cannot decide the winner:

```xml
<filters>
    <filter>
        <artifact>com.arcadedb:arcadedb-engine</artifact>
        <excludes>
            <exclude>arcadedb-log.properties</exclude>
        </excludes>
    </filter>
    <!-- ... your other filters ... -->
</filters>
```

Step 1 alone is a coin flip inside an uber-jar: two resources with the same path, and first-match wins. In an exploded classpath `target/classes` precedes `lib/*`, so it appears to work — then the same code ships as a shaded jar and silently regresses.

### Depending on a library that ships the file is NOT enough — measured, 2026-08-27

`datapotter-arcadedbhelper` 1.5 and later ship a console-only `arcadedb-log.properties`, which protects a project that inherits it. It is a strong default and it is not a guarantee, and here is the case that proves it: a project depending on datahelper **1.6** dropped a `log/` folder anyway.

The reason is dependency ORDER. Maven puts direct dependencies on the classpath in the order the pom declares them, and `getResourceAsStream` returns the first hit. That pom read:

```xml
<dependency><groupId>com.arcadedb</groupId><artifactId>arcadedb-engine</artifactId>...</dependency>
<dependency><groupId>io.github.datapotter</groupId><artifactId>datapotter-arcadedbhelper</artifactId>...</dependency>
```

The engine is declared first, so the engine's copy wins and datahelper's never gets looked at.

So the rule is:

- A project that does **not** declare `arcadedb-engine` directly is protected by the library, because direct dependencies precede transitive ones on the classpath.
- A project that **does** declare the engine directly, before the library, silently loses — and declaring the engine directly is exactly what a launched application tends to do.
- Reordering the pom works and is fragile: the next person to tidy the dependency block undoes it, with no test that fails.

**The deterministic fix for a plain, non-shaded maven project is its own `src/main/resources/arcadedb-log.properties`**, because `target/classes` always precedes the dependency jars. That is step 1 above, and it is why step 1 is written as mandatory rather than as "unless a library already covers you". For a shaded jar you still need step 2 as well.

### Where the file goes in a multi-module build

In the module that **depends on arcadedb-engine**, not the aggregator. A root pom with `<packaging>pom</packaging>` never packages `src/main/resources` at all, so a file placed there is silently inert. Put the shade filter in whichever module actually produces the runnable jar.

### Verifying, rather than assuming

```bash
# exactly one copy must survive in the shaded jar
unzip -l target/your-app-shaded.jar | grep -c arcadedb-log.properties   # must print 1

# and the real test: delete log/, run something that opens the DB, see if it comes back
rm -rf log && your-tool <some-command> && ls log 2>/dev/null || echo "clean - no log/ recreated"
```

The second check is the one that counts. The first only proves the packaging is what you wrote.

### If you are hunting existing damage

`log/` folders are not automatically junk. Some accumulated real data because a later tool found the folder and wrote into it — one held `train-*.json/tsv/xlsx`, another ~200 `assist-*.log`. Inventory before deleting; never a blanket `rm`.

---
## IMPORTANT: Two-Compilation Requirement

⚠️ **Critical:** Projects using `@ArcadeData` may require **two compilations** on first build. The first will generate the abstract (`_A`) classes source code, upon which the originating class also becomes compilable. Sometimes one compilation is enough. Point is: just because it doesn't compile once, don't assume code is broken and attempt fixes. For classes annotated with @ArcadeData you need to try at least two compilations before assuming code is broken.

### Why Two Compilations?

The sealed class pattern has a circular dependency:
1. **First compilation:** Annotation processor generates `{ClassName}_A` sealed abstract parent
2. **Second compilation:** Child class can now `extend {ClassName}_A` (which `permits` it)

### How to Handle

**During Development:**
```bash
# First time or after changing fields
mvn compile   # First compilation - generates _A class
mvn compile   # Second compilation - compiles child extending _A
```

**In CI/CD:**
```bash
# Just use clean install - Maven handles it automatically
mvn clean install
```

**In IDE (NetBeans/IntelliJ/Eclipse):**
- First build may show errors (red squiggles) - this is normal
- Run Build → Build Project twice
- Or use Maven tool window: run `compile` goal twice
- After first successful build, incremental compilation works normally

### Workflow Pattern
#### Direct pattern 
We can directly write code like this 
```java 
@ArcadeData
public final class Person extends Person_A {
    String name;
    String email;

    public static final TypeDef<Person> TYPEDEF =
        schemaBuilder().factory(Person::new).unique($email).__();
}
```
But if issues persist, you can use indirect pattern 

#### Indirect pattern only for troubleshooting code generation 
```java
// 1. Write your DTO WITHOUT extends clause first
@ArcadeData
public final class Person {
    String name;
    String email;
}

// 2. Compile once - generates Person_A

// 3. Add extends clause and imports
@ArcadeData
public final class Person extends Person_A {
    String name;
    String email;

    public static final TypeDef<Person> TYPEDEF =
        schemaBuilder().factory(Person::new).unique($email).__();
}

// 4. Compile again - success!
```

---

## Complete Model Definition Pattern

### Example: Person DTO with @ArcadeData

```java
package com.example.model;

import datapotter.arcadedbhelper.ArcadeData;
import datapotter.arcadedbhelper.TypeDef;
// import static com.example.model.Person_A.*; // because we inherit Person_A this static import is not really required. but nice to document this so that we know we need this when accessing utility static fields/methods from other classes.

@ArcadeData
public final class Person extends Person_A {

    String name;
    String email;
    Integer age;
    Long createdAt;

    public static final TypeDef<Person> TYPEDEF =
        schemaBuilder()
            .factory(Person::new)
            .unique($email)
            .lsmIndex($name)
            .__();
}
```

### What Gets Generated: Person_A

```java
// Generated by @ArcadeData processor
public abstract sealed class Person_A
    implements DataHelper_I<Person>, ArcadeDoc_I<Person>
    permits Person {

    // Field symbols for type-safe access
    public static final Field<Person, String> $name = new Field("name", String.class);
    public static final Field<Person, String> $email = new Field("email", String.class);
    public static final Field<Person, Integer> $age = new Field("age", Integer.class);
    public static final Field<Person, Long> $createdAt = new Field("createdAt", Long.class);

    // All fields list
    public static final List<Field_I<Person, ?>> FIELDS =
        List.of($name, $email, $age, $createdAt);

    // Delegation field (accesses child's package-private fields)
    private final Person sub = (Person) this;

    // Schema builder helper (pre-fills class and fields)
    public static SchemaBuilder<Person> schemaBuilder() {
        return defType(Person.class).fields(FIELDS);
    }

    // JavaBean getters/setters (delegate to child)
    public String getName() { return sub.name; }
    public void setName(String name) { sub.name = name; }
    // ... etc for all fields

    // Fluent accessors (return sub for chaining)
    public String name() { return getName(); }
    public Person name(String name) { setName(name); return sub; }
    // ... etc for all fields

    // All 15 DataHelper_I methods (generated)
    @Override
    public Object getPropertyByName(String propertyName) {
        return switch (propertyName) {
            case "name" -> getName();
            case "email" -> getEmail();
            case "age" -> getAge();
            case "createdAt" -> getCreatedAt();
            default -> null;
        };
    }
    // ... other DataHelper_I methods

    // ArcadeDoc_I methods inherited
}
```

### Benefits of Generated Code

1. **No Lombok needed** - All getters/setters generated
2. **Type-safe fields** - `$email` instead of `"email"` strings
3. **Schema helper** - `schemaBuilder()` pre-fills class and fields
4. **Fluent API** - Method chaining with proper return types
5. **Sealed class** - Strict and safe - only `Person` can extend `Person_A`

---

## Database Service Pattern

```java
package com.example.db;

import com.arcadedb.ContextConfiguration;
import com.arcadedb.GlobalConfiguration;
import com.arcadedb.database.Database;
import com.arcadedb.engine.ComponentFile;
import com.arcadedb.server.ArcadeDBServer;
import datapotter.arcadedbhelper.InitDoc;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.concurrent.atomic.AtomicBoolean;

public class DB {

    private static final String DB_NAME = "my_app_db";
    private static final String DEFAULT_ROOT_PASSWORD = "admin12345";

    private static DB instance;
    private ArcadeDBServer server;
    private final AtomicBoolean isInitialized = new AtomicBoolean(false);

    private DB() {
        init();
    }

    public static synchronized DB get() {
        if (instance == null) {
            instance = new DB();
        }
        return instance;
    }

    private void init() {
        if (isInitialized.get()) {
            return;
        }

        try {
            Path dbDir = Path.of(System.getProperty("user.home"), ".myapp", "database");
            Files.createDirectories(dbDir);

            GlobalConfiguration.SERVER_ROOT_PATH.setValue(dbDir.toString());
            GlobalConfiguration.PROFILE.setValue("low-ram");
            GlobalConfiguration.SERVER_ROOT_PASSWORD.setValue(DEFAULT_ROOT_PASSWORD);
            GlobalConfiguration.SERVER_METRICS.setValue(false);

            server = new ArcadeDBServer(new ContextConfiguration());
            server.start();

            if (!server.existsDatabase(DB_NAME)) {
                server.createDatabase(DB_NAME, ComponentFile.MODE.READ_WRITE);
            }

            Database db = server.getDatabase(DB_NAME);
            initSchema(db);

            isInitialized.set(true);
        } catch (Exception e) {
            throw new RuntimeException("Database initialization failed", e);
        }
    }


    private void initSchema(Database db) {
        // Pay attention here: How the schema is getting initialized in the db
        InitDoc.initDocTypes(db,
            Person.TYPEDEF,
            Address.TYPEDEF);
    }

    /**
     * Gets database instance - DO NOT CLOSE returned instance.
     * Server manages lifecycle.
     */
    public Database getDatabase() {
        if (!isInitialized.get()) {
            throw new IllegalStateException("Database not initialized");
        }
        return server.getDatabase(DB_NAME);
    }

    public synchronized void shutdown() {
        if (!isInitialized.getAndSet(false)) {
            return;
        }
        if (server != null && server.isStarted()) {
            try {
                server.stop();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

---

## CRUD Operations with @ArcadeData

### 1. Create and Save (Instance-Level DSL)

```java
// Create instance
Person person = new Person()
	.name("Alice")
	.email("alice@example.com")
	.age(28)
	.createdAt(System.currentTimeMillis()); // prefer fluent 

Database db = DB.get().getDatabase();

// Modern instance-level DSL (RECOMMENDED)
db.transaction(() -> {
    person.in(db)
        .whereEq($email, person.email())  // Type-safe field
        // A field which is not inside Person object cannot be used here,
        // because of type safety
        .upsert();
});

// Additionally, DB operations related to a data-type/object we can keep in the same class.
// That way field type variable like $email need not be static imported.
// The code is cleaner and IDE auto-completion happens smoothly.
// Also refactoring doesn't break the code.
// Example: following utility function can be inside Person class:

public MutableDocument upsert(Database db) {
    // If caller already has a transaction in process
    return in(db).whereEq($email, email()).upsert();
}

// Or if we must ensure cleaning up/closing of resources ourselves
// and we don't need the return MutableDocument from the db
public void upsertInTransaction(Database db) {
    db.transaction(() -> in(db).whereEq($email, email()).upsert());
}

```

### 2. Query and Load

```java
Database db = DB.get().getDatabase();

// Using native Java DSL with generated field symbols
var docs = db.select()
    .fromType(Person.__)
    .where().property($email.__).eq().value("alice@example.com")
    .documents();

// Note for full syntax for query dsl refer the ArcadeDB source (github.com/ArcadeData/arcadedb): engine/src/main/java/com/arcadedb/database/Database.java and engine/src/main/java/com/arcadedb/query/select/Select.java etc.

// Or make this a utility function in the object
public static ResultSet whereEmail(Database db, String emailId) {
    return db.select()
        .fromType(Person.__)
        .where().property($email.__).eq().value(emailId)
        .documents();
}
var docs = Person.whereEmail(db, "alice@example.com");

if (docs.hasNext()) {
    Person person = new Person();
    person.fromArcadeDocument(docs.next());

    System.out.println("Found: " + person.name());
    System.out.println("Email: " + person.email());
}
```

### 3. Update Specific Fields

```java
Database db = DB.get().getDatabase();

db.transaction(() -> {
    Update.use(db)
        .select(Person.__)
        .whereEq($email, "alice@example.com")  // Type-safe: Field_I<Person, String>
        .upsert()
        .__($age, 29)                          // Type-safe: Field_I<Person, Integer>
        .__($name, "Alice Updated")            // Type-safe: Field_I<Person, String>
        .saveDocument();
});

// Or if we have a person object (partial or whatever)
var personPartial = new Person().age(29).name("Alice Updated");
db.transaction(() -> {
    Update.use(db)
        .select(Person.__)
        .whereEq($email, "alice@example.com")  // Type-safe: Field_I<Person, String>
        .upsert()
        .from(personPartial, $age, $name)      // Type-safe: Field_I varargs
        .saveDocument();
});
```

### 4. Selective Field Updates (Instance DSL)

```java
// More terse and best when we have a person object (partial or whatever)
var person = new Person().age(29).name("Alice Updated");

// Only update these fields, not all
db.transaction(() -> {
    person.in(db)
        .whereEq($email, "alice@example.com")  // Type-safe: Field_I<Person, String>
        .from($age, $name)                      // Type-safe: Field_I varargs
        .upsert();
});
```



### 5. Find All with Ordering

```java
Database db = DB.get().getDatabase();

var docs = db.select()
    .fromType(Person.__)  // Type-safe class reference (better than "Person" string)
                          // Common AI pitfall: writing "Person" instead of Person.__
                          // Using Person.__ enables: find usage, refactor, etc.
    .orderBy($createdAt.__, false)  // Descending
                                     // Common AI pitfall: using name() instead of __ shorthand
    .documents();

List<Person> people = new ArrayList<>();
while (docs.hasNext()) {
    Person p = new Person();
    p.fromArcadeDocument(docs.next());
    people.add(p);
}
```

### 6. Typed reads (`Query`)

*New in datahelper 1.3.*

The write side has been typed for a while; reads were the one place the library still made you write string literals. `Query` is the counterpart to `Upsert`.

First, declare the type's factory once, in the model:

```java
public static final TypeDef<DriveFile> TYPEDEF =
        schemaBuilder()
                .factory(DriveFile::new)     // <- one line, once
                .unique($fileId)
                .lsmIndex($driveId)
                .__();
```

Then reads never name a type or a field as a string again:

```java
import static datapotter.arcadedbhelper.Query.query;
import static com.example.model.DriveFile_A.*;      // $fileId, $driveId, $modifiedTime

// Lazy: a Query IS an Iterable. Nothing is buffered, and break stops the scan.
for (DriveFile f : query(db, DriveFile.TYPEDEF).eq($driveId, driveId)) {
    process(f);
}

DriveFile one = query(db, DriveFile.TYPEDEF).eq($fileId, id).firstOrNull();
long folders  = query(db, DriveFile.TYPEDEF).eq($folder, true).count();

List<DriveFile> recent = query(db, DriveFile.TYPEDEF)
        .gt($modifiedTime, since)
        .orderByDesc($modifiedTime)
        .limit(20)
        .list();                               // the one eager terminal
```

**Why bother, when `fromType(DriveFile.__)` already avoids the type-name string?** Because `.__` is a `String` and binds nothing: `property($size.__).eq().value("big")` compiles cleanly against a `Long` column and simply matches nothing at runtime. `Field_I<E,T>` carries the value type, so `eq($size, "big")` is a compile error.

**Operators:** `eq neq lt le gt ge like ilike in between isNull isNotNull`. **Connectors:** conditions AND by default; `.or()` switches the next one. **Modifiers:** `orderByAsc/orderByDesc/orderBy`, `limit`, `skip`, `polymorphic`. **Terminals:** for-each, `stream()`, `firstOrNull()`, `first()`, `list()`, `count()`, `exists()`, plus `documents()` / `vertices()` / `firstVertex()` for the engine's raw records.

**Laziness, explicitly.** Everything except `list()` pulls rows as they are needed — `stream().findFirst()` stops the scan, and breaking out of a for-each stops it too (the engine's `SelectIterator.close()` is a no-op, so there is nothing to leak). Reach for `documents()` when a bulk pass reads two fields out of hundreds of thousands of rows and a DTO per row would be pure allocation.

**Two limits it inherits from the engine, and one it improves on:**

- **No parentheses, no `NOT`.** Precedence is SQL's: `a AND b OR c` means `(a AND b) OR c`. A real `(a OR b) AND (c OR d)` is inexpressible in the engine's builder — that case, and only that case, is a SQL string.
- **Nested embedded fields still cannot be filtered.** But instead of returning an empty result and letting you believe it, `Query` **throws** an `IllegalArgumentException` naming the four working alternatives. This is the one place the wrapper is strictly safer than the raw builder rather than merely tidier.

### 7. Vertex-addressed adjacency (`Traverse`)

*New in datahelper 1.3.*

`$out`/`$in`/`$both` on a snapshot address by `$rid()`, so they spend a `RID.create(db, rid).asVertex(true)` to get a vertex. That is right when you hold a DTO and wrong when you already hold the `Vertex` — a lookup by business key gives you one, and going through a snapshot would mean building a DTO purely to read its RID and then re-fetching the vertex you started from. Over a subtree walk that is a redundant record fetch per node.

```java
Vertex folder = query(db, DriveFile.TYPEDEF).eq($fileId, id).firstVertex();
List<DriveFile> children = Traverse.out(folder, DriveFile::new, ParentOf.class).getAll();
// also Traverse.in / .both / .adjacent(v, DIRECTION, ...)
```

Same traversal, same results as `$out` — just addressed the other way round.

### 8. Typed field-level upsert (`Upsert`)

*New in datahelper 1.2.*

The instance DSL writes **the whole object**, which is right when the object *is* the truth — you built it, it is complete, you mean all of it. It is wrong, quietly, when you hold a **stale snapshot** and mean to change one field: every non-null field of that snapshot gets pushed back over the row, reverting whatever else changed in between. Null fields are skipped, so the damage is partial and easy to miss in a diff.

`Upsert` is the complement: name the key, name the fields you mean, touch nothing else.

```java
import static datapotter.arcadedbhelper.Upsert.upsert;
import static com.example.model.DriveCorpus_A.*;      // $driveId, $scannedBy, $scannedAt

db.transaction(() ->
    upsert(db, DriveCorpus.TYPEDEF)
        .key($driveId, driveId)               // required — identifies the record
        .set($scannedBy, email)               // writes, including null
        .setIfPresent($scannedAt, maybeNull)  // writes only when non-null
        .save());
```

- **No string literals at the call site.** The type name comes from the `TYPEDEF`; field names come from the generated `Field_I` constants.
- **Field and value are bound by the generic signature** — putting a `String` into a `Long` field does not compile.
- **`set` vs `setIfPresent`:** `set(field, null)` erases, because "set it to nothing" is a legitimate instruction. `setIfPresent(field, null)` leaves the existing value alone — that is the one you want for data arriving from a source that sometimes omits it (a narrower API field mask, say), so a cheap refresh never erases what a fuller one collected.
- **A missing `.key(...)` throws** `IllegalStateException` rather than silently inserting.
- **Works on DOCUMENT and VERTEX types alike** (creation goes through `NewRecord`); an EDGE type is refused with a message pointing at `newEdge`.
- **Must be called inside a transaction.**

Choosing between the two: use the instance DSL when you are writing a whole object you just constructed; use `Upsert` when you are amending an existing record, and especially when the object you hold was loaded some time ago.

---

## Warning Suppression & Logback Configuration

> **This section is about console noise. It does NOT stop ArcadeDB writing a `log/` folder into your working directory — that is a separate, mandatory fix. See "stop ArcadeDB writing `log/` into whatever folder you launched from" above.**

### The Problem: Warnings from Deprecated APIs and GraalVM

When running ArcadeDB applications with GraalVM/Truffle, you'll see warnings like:

```
WARNING: A terminally deprecated method in sun.misc.Unsafe has been called
WARNING: sun.misc.Unsafe::objectFieldOffset has been called by com.oracle.truffle.api.dsl.InlineSupport$UnsafeField
WARNING: sun.misc.Unsafe::objectFieldOffset will be removed in a future release
```

And GraalVM polyglot engine warnings:
```
[engine] WARNING: The polyglot engine uses a fallback runtime that does not support runtime compilation to native code.
```

These are **deprecation warnings only** - the code works fine. They can be suppressed without affecting functionality.

### Solution 1: JVM Global Flag (Recommended for Users)

Set this environment variable globally on your system:

**Windows:**
```cmd
set JDK_JAVA_OPTIONS=--sun-misc-unsafe-memory-access=allow
```

**Linux/Mac:**
```bash
export JDK_JAVA_OPTIONS="--sun-misc-unsafe-memory-access=allow"
```

Then run your JAR normally:
```bash
java -jar your-app.jar
```

### Solution 2: Programmatic Suppression (Code Level)

In your `Main` class static initializer, suppress GraalVM polyglot warnings:

```java
public class Main {
    // Suppress GraalVM/Truffle warnings before anything else initializes
    static {
        // Suppress polyglot engine interpreter-only mode warning
        System.setProperty("polyglot.engine.WarnInterpreterOnly", "false");

        // Suppress Java Util Logging from GraalPolyglotEngine
        java.util.logging.Logger.getLogger("com.arcadedb.script.GraalPolyglotEngine")
            .setLevel(java.util.logging.Level.OFF);
        java.util.logging.Logger.getLogger("GraalPolyglotEngine")
            .setLevel(java.util.logging.Level.OFF);
        java.util.logging.Logger.getLogger("com.arcadedb")
            .setLevel(java.util.logging.Level.WARNING);
    }

    private static final Logger log = LoggerFactory.getLogger(Main.class);

    public static void main(String[] args) {
        // Your application code
    }
}
```

### Solution 3: Logback Configuration (Recommended for Clean Output)

In `src/main/resources/logback.xml`: 


```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>

    <!-- Standard ArcadeDB suppressions -->
    <logger name="com.arcadedb" level="WARN"/>
    <logger name="com.arcadedb.script.GraalPolyglotEngine" level="OFF"/>
    <logger name="com.oracle.truffle" level="ERROR"/>
    <logger name="org.graalvm" level="ERROR"/>

    <!-- Your application -->
    <logger name="com.example" level="DEBUG"/>
</configuration>
```

### Solution 4: the lines printed on EVERY exit (`Received shutdown signal`, `- Stop JMX Metrics`) — logback and levels do NOT touch them (2026-10-16)

Every JVM that starts an embedded `ArcadeDBServer` prints this to stderr when it exits, even on a perfectly normal exit, and it is not a warning about JMX:

```
Received shutdown signal. The server will be halted
- Stop JMX Metrics
```

Measured in `littlejlib/proj` (26.7.3), and the same two lines sit in the logs of every other project here that embeds the server. `SERVER_METRICS=false` does not remove it, the `com.arcadedb` logback level does not remove it, and the `--sun-misc-unsafe-memory-access` flag has nothing to do with it.

Mechanism, read from the decompiled 26.7.3 and 26.9.1 sources. `ArcadeDBServer` registers a JVM shutdown hook. The hook sets `DefaultLogger.setShuttingDown(true)` and then logs "Received shutdown signal" at SEVERE, so it reads like a crash. Once `shuttingDown` is true, `DefaultLogger.log` skips java.util.logging entirely and writes anything at INFO or above straight to `System.err`, which is why no logging configuration can reach it, and why the plugin line "- Stop JMX Metrics" comes out of the same path. Calling `server.stop()` yourself does not help, because the hook still logs its first line.

Fix: swap in a `Logger` that drops calls during shutdown unless they carry a throwable (real errors always do). Install it once, before `new ArcadeDBServer(...)`:

```java
final class QuietShutdownLogger implements com.arcadedb.log.Logger {
    final Logger delegate;
    QuietShutdownLogger(Logger delegate) { this.delegate = delegate; }

    static void install() { LogManager.instance().setLogger(new QuietShutdownLogger(new DefaultLogger())); }

    static boolean quiet(Throwable t) { return t == null && DefaultLogger.isShuttingDown(); }

    @Override public void log(Object r, Level l, String m, Throwable t, String c, Object[] a) {
        if (!quiet(t)) delegate.log(r, l, m, t, c, a);
    }
    @Override public void log(Object r, Level l, String m, Throwable t, String c,
            Object a1, Object a2, Object a3, Object a4, Object a5, Object a6, Object a7, Object a8, Object a9,
            Object a10, Object a11, Object a12, Object a13, Object a14, Object a15, Object a16, Object a17) {
        if (!quiet(t)) delegate.log(r, l, m, t, c, a1, a2, a3, a4, a5, a6, a7, a8, a9, a10, a11, a12, a13, a14, a15, a16, a17);
    }
    @Override public void flush() { delegate.flush(); }
}
```

Traps: (i) 26.7.3 has no `LogManager.getLogger()` (26.9.1 does), so wrap a fresh `new DefaultLogger()` rather than reading the current one, which compiles on both; (ii) the `Logger` interface has TWO `log` overloads, one with 17 trailing `Object` parameters, and a wrapper that implements only the array one does not compile; (iii) `setLogger` is the supported hook, the `arcadedb.log.impl` property only accepts `default` and `slf4j`. Working class: `littlejlib/proj` `db/QuietShutdownLogger`.

---

## Complete Working Example

This section provides a complete, runnable example using the @ArcadeData pattern with all best practices applied.

### 1. Model Classes

```java
package com.example.model;

import datapotter.arcadedbhelper.ArcadeData;
import datapotter.arcadedbhelper.TypeDef;
import static com.example.model.Person_A.*;

/**
 * Person entity with @ArcadeData annotation.
 * Parent class Person_A is generated with all accessors and field symbols.
 */
@ArcadeData
public final class Person extends Person_A {

    String name;
    String email;
    Integer age;
    Long createdAt;
    String address;

    public static final TypeDef<Person> TYPEDEF =
        schemaBuilder()
            .factory(Person::new)
            .unique($email)
            .__();
}
```

```java
package com.example.model;

import datapotter.arcadedbhelper.ArcadeData;
import datapotter.arcadedbhelper.TypeDef;
import static com.example.model.Address_A.*;

@ArcadeData
public final class Address extends Address_A {

    String street;
    String city;
    String zipCode;

    public static final TypeDef<Address> TYPEDEF =
        schemaBuilder().factory(Address::new).__();
}
```

### 2. Database Service (Singleton Pattern)

```java
package com.example.db;

import com.arcadedb.ContextConfiguration;
import com.arcadedb.GlobalConfiguration;
import com.arcadedb.database.Database;
import com.arcadedb.engine.ComponentFile;
import com.arcadedb.server.ArcadeDBServer;
import datapotter.arcadedbhelper.InitDoc;
import com.example.model.Person;
import com.example.model.Address;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.concurrent.atomic.AtomicBoolean;
import static com.arcadedb.GlobalConfiguration.*;

public class DB {

    private static final String DB_NAME = "example_db";
    private static final String DEFAULT_ROOT_PASSWORD = "admin12345";

    private static DB instance;
    private ArcadeDBServer server;
    private final AtomicBoolean isInitialized = new AtomicBoolean(false);

    private DB() {
        init();
    }

    public static synchronized DB get() {
        if (instance == null) {
            instance = new DB();
        }
        return instance;
    }

    private void init() {
        if (!isInitialized.compareAndSet(false, true)) {
            return;
        }

        try {
            var dbPath = Path.of(System.getProperty("user.home"), "org_name","project_name", DB_NAME);
				// for DB storage location follow actual project conventions and specifications
				// this is just an example here
            boolean isNewDatabase = !Files.exists(dbPath);

            var config = new ContextConfiguration();
            config.setValue(SERVER_ROOT_PASSWORD, DEFAULT_ROOT_PASSWORD);
            config.setValue(SERVER_DATABASE_DIRECTORY, dbPath.getParent().toString());
            config.setValue(HA_ENABLED, false);

            server = new ArcadeDBServer(config);
            server.start();

            if (isNewDatabase) {
                server.createDatabase(DB_NAME, ComponentFile.MODE.READ_WRITE);
            }

            Database db = getDatabase();
            initSchema(db);

            System.out.println("Database initialized: " + DB_NAME);

        } catch (Exception e) {
            throw new RuntimeException("Failed to initialize database", e);
        }
    }

    private void initSchema(Database db) {
        // DO NOT CLOSE db - server manages lifecycle
        InitDoc.initDocTypes(db,
            Person.TYPEDEF,
            Address.TYPEDEF
        );
        // DO NOT CLOSE db - server manages lifecycle
    }

    /**
     * Gets database instance.
     * CRITICAL: DO NOT CLOSE the returned Database instance!
     * The server manages the database lifecycle.
     */
    public Database getDatabase() {
        return server.getDatabase(DB_NAME);
    }

    public void shutdown() {
        if (server != null) {
            server.stop();
        }
    }
}
```

### 3. Main Application with All CRUD Operations

```java
package com.example;

import com.arcadedb.database.Database;
import com.arcadedb.query.sql.executor.ResultSet;
import com.example.db.DB;
import com.example.model.Person;
import datapotter.arcadedbhelper.Update;

import java.util.ArrayList;
import java.util.List;

import static com.example.model.Person_A.*; // this is very important, don't forget such lines
// sometimes two dataclasses might have fields with same name, example uuid, or name 
// then this static import may conflict. 
// Two solutions;
// (i) use explicit reference/name when conflict happens :  Person_A.$name or Person.$name (both work) will not conflict with City_A.$name or City.$name 
// (ii) all operations make them like utility function and export out of the dataclass, this way u avoid conflicts, similar logic is together, static imports are not needed, refactoring is easier because everything related to the dataclass is pretty much in the same place 
// Alternatively don't bother too much until errors come, and if errors fix them, it would be easy.

public class Main {

    // Suppress GraalVM/Truffle warnings
    static {
        System.setProperty("polyglot.engine.WarnInterpreterOnly", "false");
        java.util.logging.Logger.getLogger("com.arcadedb.script.GraalPolyglotEngine")
            .setLevel(java.util.logging.Level.OFF);
        java.util.logging.Logger.getLogger("com.arcadedb")
            .setLevel(java.util.logging.Level.WARNING);
    }

    public static void main(String[] args) {
        // DO NOT use try-with-resources for Database!
        // Server manages database lifecycle
        var db = DB.get().getDatabase();

        try {
            createPersons(db);
            readPersons(db);
            updatePerson(db);
            selectiveUpdate(db);
            queryWithOrdering(db);
        } finally {
            // Shutdown server, not individual database
            DB.get().shutdown();
        }
    }

    static void createPersons(Database db) {
        System.out.println("\n=== CREATE ===\n");

        var alice = new Person()
            .name("Alice Johnson")
            .email("alice@example.com")
            .age(28)
            .createdAt(System.currentTimeMillis());

        db.transaction(() -> alice.in(db).upsert());
        System.out.println("Created: " + alice.name());

        db.transaction(() -> {
            Update.use(db)
                .select(Person.__)
                .whereEq($email, "bob@example.com")
                .upsert()
                .__($name, "Bob Smith")
                .__($age, 35)
                .__($createdAt, System.currentTimeMillis())
                .saveDocument();
        });
        System.out.println("Created: Bob Smith");
    }

    static void readPersons(Database db) {
        System.out.println("\n=== READ ===\n");

        var docs = db.select()
            .fromType(Person.__)
            .where().property($email.__).eq().value("alice@example.com")
            .documents();

        if (docs.hasNext()) {
            var person = Person.of(docs.next());  // Static factory from generated _A class
            System.out.println("Found: " + person.name() + ", age: " + person.age());
        }
    }

    static void updatePerson(Database db) {
        System.out.println("\n=== UPDATE ===\n");

        db.transaction(() -> {
            Update.use(db)
                .select(Person.__)
                .whereEq($email, "alice@example.com")
                .upsert()
                .__($age, 29)
                .__($name, "Alice Updated")
                .saveDocument();
        });
        System.out.println("Updated Alice");
    }

    static void selectiveUpdate(Database db) {
        System.out.println("\n=== Selective Update ===\n");

        Person partial = new Person().age(36).address("123 New Street");
        db.transaction(() -> {
            Update.use(db)
                .select(Person.__)
                .whereEq($email, "bob@example.com")
                .upsert()
                .from(partial, $age, $address)
                .saveDocument();
        });
        System.out.println("Updated Bob's age and address");
    }

    static void queryWithOrdering(Database db) {
        System.out.println("\n=== Query with Ordering ===\n");

        var docs = db.select()
            .fromType(Person.__)
            .orderBy($createdAt.__, false)
            .documents();

        var people = new ArrayList<Person>();
        while (docs.hasNext()) {
            people.add(Person.of(docs.next()));  // Static factory — one line per object
        }

        System.out.println("Found " + people.size() + " persons");
        people.forEach(p ->
            System.out.println("  - " + p.name() + " (" + p.email() + "), age: " + p.age())
        );
    }
}
```

### Key Patterns Demonstrated

✅ **Type-Safe API:** `.whereEq($email, value)`, `.__($age, 29)`, `.from(obj, $age, $name)`
✅ **Instance DSL:** `person.in(db).upsert()`
✅ **Static DSL:** `Update.use(db).select(Person.__)`
✅ **Transactions:** All writes wrapped in `db.transaction(() -> {...})`
✅ **Field Symbols:** `Person.__` (class name), `$email` (Field_I), `$email.__` (ArcadeDB native only)

---

## Production Essentials

### 1. Maven Configuration for Executable JAR

ArcadeDB + GraalVM requires special Maven Shade configuration:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals><goal>shade</goal></goals>
            <configuration>
                <!-- Remove conflicting JAR signatures -->
                <filters>
                    <filter>
                        <artifact>*:*</artifact>
                        <excludes>
                            <exclude>META-INF/*.SF</exclude>
                            <exclude>META-INF/*.DSA</exclude>
                            <exclude>META-INF/*.RSA</exclude>
                        </excludes>
                    </filter>
                </filters>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.example.Main</mainClass>
                        <manifestEntries>
                            <Enable-Native-Access>ALL-UNNAMED</Enable-Native-Access>
                            <Add-Opens>java.base/sun.nio.ch java.base/java.nio.channels.spi</Add-Opens>
                            <Add-Exports>jdk.unsupported/sun.misc</Add-Exports>
                        </manifestEntries>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```

**Why needed:**
- `Enable-Native-Access`: JNI performance optimization
- `Add-Opens`: NIO operations and channel management
- `Add-Exports`: Truffle DSL compilation

---

### 2. Query API Options — DSL FIRST

> **⚠️ DIRECTIVE FOR AI AGENTS — read this before writing any query. Updated for datahelper 1.3.**
>
> There are now **three** read routes, and they rank in this order:
>
> 1. **`Query` (datahelper 1.3+) — the default.** `query(db, Person.TYPEDEF).eq($email, x).firstOrNull()`. See [§6 Typed reads](#6-typed-reads-query). It wraps the native DSL below, so it inherits every advantage listed here *and adds the one thing the native DSL cannot give*: `Field_I<E,T>` binds field to value, so `eq($age, "old")` against an `Integer` column is a **compile error**. It is also lazy — a `Query` is an `Iterable`.
> 2. **Native `db.select()` — when `Query` doesn't reach.** Compiled/parameterized reuse (`compile()` + `parameter()`), `fromBuckets`, vector search, or when you want the raw `Document`/`Vertex` and no per-row object. `Query.documents()` / `.vertices()` hand you that iterator directly, so you rarely need to start from `db.select()` yourself.
> 3. **`db.query("sql", ...)` — last resort.** Defaulting to SQL strings out of training habit is the single most common mistake when generating ArcadeDB code with this library. Use it only for what neither route above can express: aggregation (`SUM`/`GROUP BY`), `(a OR b) AND (c OR d)` grouping, `MATCH`/`TRAVERSE`, and nested-field filtering. **Always parameterize with `?`; never concatenate.**
>
> If your first instinct was a SQL `SELECT`, stop. If your second was `db.select().fromType("Person").where().property("email")`, stop again — that is the string-literal form `Query` exists to replace.

**The rest of this section describes route 2, the native DSL.** It remains worth understanding because `Query` is a thin wrapper over it and its limits are `Query`'s limits.

**Why the DSL is the default (not SQL strings):**
- **Faster** — no SQL parser runs; the query tree executes directly. The engine's own javadoc: *"could be much faster than the same SQL query because it does not use any parser."*
- **Index-aware** — it transparently picks LSM/unique indexes. Confirm with `iterator.getMetrics().get("usedIndexes")`.
- **Refactor-safe** — type and field names come from generated constants (`Person.__`, `$email.__`), so a rename breaks the call site at **compile time**. A `"SELECT ... WHERE email ..."` string silently rots when a field is renamed.
- **Injection-safe** — values are bound (`.value(x)` / `.parameter(n)`), never concatenated into a query string.
- **Structurally checked** — the fluent state-machine won't even compile a malformed query; a SQL typo only blows up at runtime.

**Native DSL (DEFAULT — use this):**
```java
// Returns SelectIterator<Document> - executes directly, no SQL parsing
var docs = db.select().fromType(Person.__)
    .where().property($email.__).eq().value("alice@example.com")
    .documents();
```

**SQL / language query (ONLY when the DSL can't express it):**
```java
// Returns ResultSet - SQL/Gremlin/Cypher. ALWAYS parameterize with ?, never concatenate values.
try (var rs = db.query("sql", "SELECT FROM Person WHERE email = ?", "alice@example.com")) {
    while (rs.hasNext()) {
        var person = Person.of(rs.next().toElement().asDocument());  // back to a typed object
        // process
    }
}
```

**Use a SQL string ONLY for the things the DSL genuinely lacks:**
- **Aggregations** — `sum`, `avg`, `min`, `max`, `group by`, `having`, `distinct`. (The DSL has only `count()` and `exists()`.)
- **Projections / computed columns** — `SELECT name, age*2 ...`, or nested link projection `SELECT *, customer:{*} ...`. (The DSL always returns whole records.)
- **Nested / dot-path fields in WHERE or ORDER BY** — e.g. filtering on `homeAddress.city`. (DSL `property()` is a **flat** single-key lookup — no dot-paths. See the PRP-21 caution.)
- **Arbitrary parenthesized boolean logic** — `(a OR b) AND (c OR d)`. (The DSL flattens with fixed `AND`-over-`OR` precedence — see §3.)
- **SQL functions, subqueries, or joins across types.**

For **everything else** — single-type retrieval with any combination of equality/range/`in`/`like`/null conditions, ordering, paging, counts, existence checks, and graph hops — **use the DSL.**

---

### 3. Native Java DSL API Reference (complete)

The DSL is a **state machine** — each call narrows what you can call next, so let the IDE's autocomplete guide you. The build order is always: **from → where (conditions chained with `and`/`or`) → orderBy/limit/skip → terminal** (or `compile()`).

**Source / FROM:**
```java
db.select().fromType(Person.__)                 // by type name (Person.__ is the generated "Person" constant)
db.select().fromBuckets("bucket1", "bucket2")   // by specific buckets (String... or Integer...)
```

**Operators** — call after `.property($field.__)`. Binary ops then take a right operand `.value(v)` / `.parameter(n)` / `.property(otherField.__)`:
```java
.eq()  .neq()  .lt()  .le()  .gt()  .ge()    // comparisons
.like()  .ilike()                            // SQL % wildcard; ilike = case-insensitive (String operands)
.in()                                        // membership → .value(List.of(...)) or .parameter(n)
.between()                                   // INCLUSIVE range → .values(low, high)   (note: values(), plural)
.isNull()  .isNotNull()                      // null checks → NO right operand, chain straight on
```

**Chaining conditions — `.and()` / `.or()`** (the only two connectors; there is no `.not()` and no parentheses):
```java
// AND binds tighter than OR, exactly like SQL:   a AND b OR c  ==>  (a AND b) OR c
var adults = db.select().fromType(Person.__)
    .where().property($age.__).ge().value(18)
        .and().property($age.__).lt().value(65)
        .or().property($name.__).eq().value("admin")
    .documents();
// Need true (a OR b) AND (c OR d) grouping? The DSL can't express it — use a SQL string for that case only.
```

**Modifiers** — call AFTER all conditions. They return to `Select`, so you cannot call `.and()`/`.or()` again once you've used them:
```java
.orderBy($createdAt.__, false)   // false = DESC, true = ASC; call again to add secondary sort keys
.limit(20)
.skip(40)                        // skip + limit = pagination
.polymorphic(false)              // exclude subtypes (default true)
```

**Terminals** — execute the query:
```java
.documents()   // SelectIterator<Document>
.vertices()    // SelectIterator<Vertex>   — for @ArcadeData(type = VERTEX)
.edges()       // SelectIterator<Edge>     — for @ArcadeData(type = EDGE)
.count()       // long      — honors limit()
.exists()      // boolean
.stream()      // Stream<Document>
```

**Consuming a `SelectIterator<T>`:**
```java
var it = db.select().fromType(Person.__).where().property($email.__).eq().value(email).documents();
it.nextOrNull();     // first result or null — no NoSuchElementException risk (preferred over hasNext()/next() for single-row)
it.toList();         // List<T>
it.stream();         // Stream<T>
it.getMetrics();     // Map — e.g. getMetrics().get("usedIndexes") to verify an index was used
while (it.hasNext()) { Person p = Person.of(it.next()); /* ... */ }   // classic loop
```

**Compiled + parameterized (reuse the plan, rebind values — great for loops/hot paths):**
```java
SelectCompiled q = db.select().fromType(Person.__)
    .where().property($email.__).eq().parameter("email")   // .parameter(name) declares a placeholder
    .compile();

var alice = q.parameter("email", "alice@example.com").documents().nextOrNull();  // .parameter(name, value) binds it
var bob   = q.parameter("email", "bob@example.com").documents().nextOrNull();
// .value(x) bakes a literal into the plan; .parameter("n") is a placeholder bound later on the compiled query.
```

---

### 4. Query Operations Examples

**Find by Field:**
```java
public Optional<Person> findByEmail(String email) {
    var docs = db.select().fromType(Person.__)
        .where().property($email.__).eq().value(email)
        .documents();

    if (!docs.hasNext()) return Optional.empty();
    return Optional.of(Person.of(docs.next()));  // Static factory
}
```

**Find by Multiple Conditions (AND / OR) — do NOT drop to SQL for this:**
```java
// "active adults named like 'jo%'"  — entirely expressible in the DSL
public List<Person> findActiveAdults() {
    return db.select().fromType(Person.__)
        .where().property($age.__).ge().value(18)
            .and().property($age.__).lt().value(65)
            .and().property($name.__).ilike().value("jo%")
        .documents()
        .stream().map(Person::of).toList();
}
```

**Range, IN, and NULL checks:**
```java
// BETWEEN is inclusive on both ends; IN takes a Collection; isNull/isNotNull take no value
var docs = db.select().fromType(Person.__)
    .where().property($age.__).between().values(25, 40)
        .and().property($email.__).isNotNull()
        .and().property($name.__).in().value(List.of("Alice", "Bob", "Carol"))
    .documents();
```

**Count / Exists (cheap aggregate terminals — no SQL needed):**
```java
long adults     = db.select().fromType(Person.__).where().property($age.__).ge().value(18).count();
boolean anyBob  = db.select().fromType(Person.__).where().property($name.__).eq().value("Bob").exists();
```

**Find All with Ordering:**
```java
public List<Person> findAllByAge() {
    var docs = db.select().fromType(Person.__)
        .orderBy($age.__, false)  // Descending
        .documents();

    var people = new ArrayList<Person>();
    while (docs.hasNext()) {
        people.add(Person.of(docs.next()));  // Static factory
    }
    return people;
}
```

**Delete Operation (DSL):**
```java
public void deleteByEmail(String email) {
    db.transaction(() -> {
        var docs = db.select().fromType(Person.__)
            .where().property($email.__).eq().value(email)
            .documents();
        if (docs.hasNext()) {
            db.deleteRecord(docs.next());
        }
    });
}
```

**Delete Operation (SQL form — shown only for contrast; prefer the DSL form above):**
```java
// The DSL version above already does this. Use SQL here ONLY if you also need a
// SQL-only feature (e.g. a dot-path/nested filter or an aggregate) in the same query.
public void deleteByEmailSQL(String email) {
    db.transaction(() -> {
        try (var rs = db.query("sql", "SELECT FROM Person WHERE email = ?", email)) {
            if (rs.hasNext()) {
                var record = rs.next().getRecord();
                record.ifPresent(db::deleteRecord);
            }
        }
    });
}
```

**Pagination:**
```java
public List<Person> findPage(int page, int pageSize) {
    var docs = db.select().fromType(Person.__)
        .orderBy($createdAt.__, false)
        .skip(page * pageSize)
        .limit(pageSize)
        .documents();

    // Convert to list
}
```

**Graph Traversal (DSL) — query then hop, no Gremlin/SQL needed:**
```java
// For @ArcadeData(type = VERTEX) types. Run a select to a vertex iterator, then traverse.
// outVertices/inVertices/bothVertices (+ thenOut.../thenIn... for multi-hop); edge types are optional varargs.
List<Vertex> alicesFriends = db.select().fromType(Person.__)
    .where().property($email.__).eq().value("alice@example.com")
    .vertices()
    .outVertices(Knows.__)              // Knows.__ == "Knows"; omit args to follow all edge types
    .toList();

// Friends-of-friends (2 hops), names only:
Set<String> fof = db.select().fromType(Person.__)
    .where().property($name.__).eq().value("Alice")
    .vertices()
    .outVertices(Knows.__)
    .thenOutVertices(Knows.__)
    .stream().map(v -> v.getString("name")).collect(Collectors.toSet());
```

**When you DO need SQL — aggregation (the DSL has no `sum`/`avg`/`group by`):**
```java
// Legitimate SQL: there is no DSL equivalent for GROUP BY / AVG.
try (var rs = db.query("sql",
        "SELECT city, count(*) AS n, avg(age) AS avgAge FROM Person GROUP BY city ORDER BY n DESC")) {
    while (rs.hasNext()) {
        var row = rs.next();
        System.out.println(row.getProperty("city") + ": " + row.<Long>getProperty("n"));
    }
}
```

---

### 5. Transaction Strategy & Performance

**Modern Transaction Patterns (ArcadeDB 25.7+):**

```java
// Lambda-based (Preferred) - automatic commit/rollback
db.transaction(() -> {
    person.in(db).upsert();
    address.in(db).upsert();
});

// Async pattern for bulk operations
database.async().transaction(() -> {
    // Background processing
});

// Traditional (complex error handling only)
db.begin();
try {
    // operations
    db.commit();
} catch (Exception e) {
    db.rollback();
    throw e;
}
```

**When to Use:**
- **Lambda**: Standard CRUD operations
- **Async**: Bulk operations, background processing
- **Traditional**: Complex error handling, multi-step validation

**Performance Tips:**
- Batch operations in single transaction
- Use `SelectCompiled` for repeated queries
- Prefer `limit()` and `skip()` for pagination
- Use indexes (unique, lsmIndex) in schema

---

### 6. Serialization Architecture

**Deserialization Patterns:**
```java
// OPTION 1: Static factory (RECOMMENDED — concise, one-liner)
var person = Person.of(doc);              // doc is a Document from native DSL

// OPTION 2: Instance method (when reusing objects or from SQL ResultSet)
var person = new Person();
person.fromArcadeDocument(doc);           // Populates all fields from Document

// For SQL queries returning ResultSet:
var person = Person.of(rs.next().toElement().asDocument());
```

**Generated deserialization (via ArcadeDoc_I):**
- `of(Document)` — static factory, creates new instance + calls fromArcadeDocument
- `fromArcadeDocument(Document)` — instance method, populates fields from Document
- Automatic type conversion
- Handles nested collections (via DataField/ListDataField)
- Null-safe field mapping

**Serialization Best Practices:**
- Use `HashMap` (ArcadeDB compatible)
- Handle nulls explicitly (empty string vs null)
- Recursive mapping for nested objects
- Return fallback objects, not null

---

### 7. Performance Optimization

**Query Performance:**
- **Compiled Queries**: `SelectCompiled` for frequent queries
- **Indexes**: Strategic `unique()` and `lsmIndex()` placement
- **Streaming**: Use iterators, avoid collecting large sets
- **Pagination**: `limit()` and `skip()` for large datasets

**Memory Management:**
- **ResultSet**: Always use try-with-resources
- **Transactions**: Batch operations in single transaction
- **Database**: Never close (server-managed)

**Index Strategy:**
```java
public static final TypeDef<Person> TYPEDEF =
    schemaBuilder()
        .factory(Person::new)     // zero-reflection factory, so reads are query(db, Person.TYPEDEF)
        .unique($email)           // Unique constraint + index
        .lsmIndex($name)          // LSM index for queries
        .__();
```

**Bulk Operations:**
```java
db.transaction(() -> {
    people.forEach(person -> person.in(db).upsert());
});
```

---

## Type-Safe vs String-Based API Reference

Our wrapper provides **type-safe Field_I versions** wherever possible. Use these for compile-time checking. String-based versions exist as fallback for ArcadeDB's native API or when Field_I not available.

### When to Use Each

**Use Type-Safe Field_I (`$field`):**
- ✅ `.whereEq($email, value)` - compile-time type checking
- ✅ `.__($age, value)` - type-safe field updates
- ✅ `.from(object, $age, $name)` - type-safe field selection
- ✅ Refactor-safe, find-usage works, IDE autocomplete

**Use String-based (`$field.__` or string literals):**
- ✅ ArcadeDB native API *when you are not going through `Query`*: `.property($email.__)`, `.orderBy($email.__)`
- ✅ Reading raw fields off a `Document` in a bulk pass: `d.getString($name.__)`
- ✅ SQL/query builders requiring strings
- ✅ Dynamic field names not known at compile time

> **Note (1.3):** `.__` is refactor-safe but **binds no type** — it is a bare `String`, so `.property($age.__).eq().value("old")` compiles against an `Integer` column and silently matches nothing. Reads should go through [`Query`](#6-typed-reads-query), where `eq($age, "old")` is a compile error. The rows below marked "N/A" for the native API are the gap `Query` closes.

### API Comparison Table

| Operation | Type-Safe (Preferred) | String-Based (Fallback) | Notes |
|-----------|----------------------|-------------------------|-------|
| **Update.whereEq()** | `.whereEq($email, "x")` | `.whereEq($email.__, "x")` | Type-safe checks String type |
| **Update.__()** | `.__($age, 29)` | `.__($age.__, () -> 29)` | Type-safe checks Integer type |
| **Update.from()** | `.from(obj, $age, $name)` | `.from(obj, $age.__, $name.__)` | Type-safe Field_I[] varargs |
| **ArcadeDoc.whereEq()** | `.whereEq($email, "x")` | `.whereEq($email.__, "x")` | Instance DSL type-safe |
| **Filter a field** | `query(db,TYPEDEF).eq($email,"x")` | `.property($email.__).eq().value("x")` | 1.3: Query binds field to value; `.__` does not |
| **Order results** | `query(db,TYPEDEF).orderByDesc($age)` | `.orderBy($age.__, false)` | 1.3: Query takes the Field_I directly |
| **Choose the type** | `query(db, Person.TYPEDEF)` | `.fromType(Person.__)` / `.fromType("Person")` | 1.3: TYPEDEF also carries the row factory |

### Example: Both Versions Side-by-Side

```java
// ========================================
// Type-Safe API (RECOMMENDED)
// ========================================
db.transaction(() -> {
    Update.use(db)
        .select(Person.__)                        // Class name (static __)
        .whereEq($email, "alice@example.com")     // Field_I + type check ✅
        .upsert()
        .__($age, 29)                             // Field_I + Integer ✅
        .__($name, "Alice")                       // Field_I + String ✅
        .saveDocument();
});

// ========================================
// String-Based API (FALLBACK)
// ========================================
db.transaction(() -> {
    Update.use(db)
        .select("Person")                         // String literal
        .whereEq("email", "alice@example.com")    // String + Object (no type check)
        .upsert()
        .__("age", () -> 29)                      // String key, no type safety
        .__("name", () -> "Alice")                // String key, no type safety
        .saveDocument();
});

// ========================================
// Mixed: Type-Safe Wrapper + ArcadeDB Native
// ========================================
var docs = db.select()
    .fromType(Person.__)                          // Our wrapper: Class __ (String)
    .where().property($email.__).eq().value("alice@example.com")  // ArcadeDB native needs String
    .orderBy($createdAt.__, false)                // ArcadeDB native needs String
    .documents();

while (docs.hasNext()) {
    var person = Person.of(docs.next());          // Static factory from generated _A
    System.out.println(person.name());
}
```

### Key Insight

The `__` field serves **two distinct purposes**:

1. **Class level** (`Person.__`): Static String field = class name
   ```java
   .fromType(Person.__)  // Inherited static field, value = "Person"
   ```

2. **Field level** (`$email.__`): Instance String field = field name
   ```java
   .property($email.__)  // Field instance's __ field, value = "email"
   ```

**Golden Rule:** Use `$field` (Field_I object) for our type-safe API. Only use `$field.__` (String) when:
- ArcadeDB's native API requires String (`.property()`, `.orderBy()`)
- No Field_I overload exists
- Dynamic/runtime field selection needed

---

## Best Practices Summary

### ✅ DO:

1. Use `@ArcadeData` for new projects (no Lombok needed)
2. Use package-private fields (omit `private` keyword)
3. Use `schemaBuilder()` helper for schema (pre-filled)
4. **Use type-safe Field_I** (`$email`) for our wrapper APIs
5. Use `$field.__` only for ArcadeDB native APIs (`.property()`, `.orderBy()`)
6. Use instance DSL (`person.in(db)`) for saves
7. Run `mvn compile` twice on first build
8. Add `final` to child class for sealed pattern

### ❌ DON'T:

1. Don't use `@Data` or Lombok with `@ArcadeData`
2. Don't make fields `private` (parent can't access)
3. **NEVER close Database instances** (server manages them)
4. **NEVER use try-with-resources for Database** (throws UnsupportedOperationException)
5. Don't forget two-compilation requirement
6. **Don't use `$field.__` when Field_I version exists** (loses type safety)
7. Don't use string literals for field names when Field_I available

---

## ⚠️ CRITICAL: Database Lifecycle Management

### The Problem: AI Training Data Anti-Pattern

Claude Code's default training includes patterns for JDBC and other databases where you close connections. **This does NOT apply to embedded ArcadeDB!**

### ❌ WRONG: Closing Database (Common AI Mistake)

```java
// WRONG - throws UnsupportedOperationException
try (Database db = DB.get().getDatabase()) {
    db.transaction(() -> {
        // operations
    });
} // Crash! Cannot close server-managed database
```

### ✅ CORRECT: Server-Managed Lifecycle

```java
// CORRECT - server manages database lifecycle
Database db = DB.get().getDatabase();

try {
    db.transaction(() -> {
        // operations
    });
} finally {
    // Shutdown server, not database
    DB.get().shutdown();
}
```

### Why This Matters

**Embedded ArcadeDB Server Pattern:**
- ✅ Server creates and manages database instances
- ✅ Database objects are server-managed references
- ✅ Only shutdown the server, never individual databases
- ❌ Calling `database.close()` throws `UnsupportedOperationException`

**When to Close:**
- ✅ Close `ResultSet` (use try-with-resources)
- ✅ Shutdown `ArcadeDBServer` on application exit
- ❌ NEVER close `Database` instances

**Note to AI Agents:**
This is an **embedded database server**, not a connection pool. The server manages all database lifecycle. Your training on JDBC/connection-based databases does not apply here.

---

## Resources

- **Generated Source Location:** `target/generated-sources/annotations/`
- **Processor Module:** `datapotter-arcadedbhelper-processor`
- **Annotation Module:** `datapotter-arcadedbhelper`
- **Base Interfaces:** `datapotter-datahelper-base`

---

---

## New Features (2026-01-14)

### PRP-21: Zero-Reflection Nested Types

**Feature:** Nested DataHelper objects (embedded types) now supported with zero reflection.

**Define nested types:**
```java
@ArcadeData
public final class Employee extends Employee_A {
    String employeeId;
    String firstName;
    String lastName;
    String email;
    Integer age;

    // Single nested object
    Address homeAddress;
    Address workAddress;

    // List of nested objects
    List<PhoneNumber> phoneNumbers;

    public static final TypeDef<Employee> TYPEDEF =
        schemaBuilder()
            .factory(Employee::new)
            .unique($employeeId)
            .lsmIndex($email)
            .__();
}

@ArcadeData
public final class Address extends Address_A {
    String street;
    String city;
    String zipCode;

	// This is NOT needed, when you register Employee schema, address schema is automatically generated using Fields static information and address is also registered !
    // public static final TypeDef<Address> TYPEDEF = schemaBuilder().factory(Address::new).__();
}

@ArcadeData
public final class PhoneNumber extends PhoneNumber_A {
    String type;
    String number;

	// NOT needed, already explained above. But documented to make a point to AI (like claude-code) that please do not blindly auto-generate this when note needed.  
    // public static final TypeDef<PhoneNumber> TYPEDEF = schemaBuilder().__();
}
```

**Generated code (automatic):**
```java
// In Employee_A.java
public static final DataField<Employee, Address> $homeAddress =
    new DataField("homeAddress", Address.class, Address_A.FIELDS);

public static final DataField<Employee, Address> $workAddress =
    new DataField("workAddress", Address.class, Address_A.FIELDS);

public static final ListDataField<Employee, PhoneNumber> $phoneNumbers =
    new ListDataField("phoneNumbers", PhoneNumber.class, PhoneNumber_A.FIELDS);
```

**Schema registration (automatic dependency detection):**
```java
// Only pass top-level type
InitDoc.initDocTypes(db, Employee.TYPEDEF);

// Address and PhoneNumber automatically detected and registered first
// Result: Address → PhoneNumber → Employee (dependency order)
```

**Runtime usage:**
```java
db.transaction(() -> {
    // Create nested objects
    var homeAddr = new Address();
    homeAddr.street("123 Main St");
    homeAddr.city("San Francisco");
    homeAddr.zipCode("94101");

    var phone1 = new PhoneNumber();
    phone1.type("mobile");
    phone1.number("555-1234");

    var phone2 = new PhoneNumber();
    phone2.type("work");
    phone2.number("555-5678");

    // Create employee with nested objects
    var employee = new Employee();
    employee.employeeId("EMP001");
    employee.firstName("Alice");
    employee.lastName("Johnson");
    employee.email("alice@example.com");
    employee.age(30);
    employee.homeAddress(homeAddr);
    employee.phoneNumbers(List.of(phone1, phone2));

    // Save (nested objects saved as ImmutableEmbeddedDocument)
    Update.use(db)
        .select("Employee")
        .whereEq($employeeId.__, employee.employeeId())
        .upsert()
        .from(employee, new String[0])
        .saveDocument();

    // Load and deserialize (concise static factory)
    var docs = db.select()
        .fromType("Employee")
        .where().property($employeeId.__).eq().value("EMP001")
        .documents();

    var employee = Employee.of(docs.next());  // Static factory method

    // Access nested objects
    System.out.println(employee.homeAddress().city());  // San Francisco
    System.out.println(employee.phoneNumbers().get(0).number());  // 555-1234
});
```

**Type-safe field chaining (for building nested paths):**

> **⚠️ Correction — nested-field FILTERING does NOT work through the native `db.select()` DSL, and fails *silently*.** The DSL's WHERE evaluator is a **flat** single-key lookup. Every terminal (`documents()`, `vertices()`, `count()`, `exists()`) re-evaluates the condition per record via `SelectExecutor.evaluateWhere()` → `SelectPropertyValue.eval()` → `record.get("homeAddress.city")` (verified in `com.arcadedb.query.select.*`). The record's map only has the key `"homeAddress"` (holding an embedded document), so the lookup returns `null`, `null == "San Francisco"` is false, and **every row is rejected — the query returns empty, no error.** Critically, **an index on the dotted path does not save it**: ArcadeDB *does* support a `homeAddress.city` index and the executor will use it to find candidate RIDs, but it then re-filters each candidate with the same flat lookup (`SelectExecutor.java` re-evaluates even after an index hit) and discards them all. SQL, by contrast, resolves dot-paths (and link hops) in its own evaluator and can use that dotted index for real — so **filtering by an embedded or referenced sub-field must go through SQL** (one of the documented DSL exceptions in §2):
> ```java
> try (var rs = db.query("sql",
>         "SELECT FROM Employee WHERE homeAddress.city = ?", "San Francisco")) {
>     while (rs.hasNext()) { var e = Employee.of(rs.next().toElement().asDocument()); /* ... */ }
> }
> ```
> Read the chaining examples below as *"how to build the `homeAddress.city` path string type-safely"* (for SQL, index defs, and updates) — **not** as a working native-DSL filter. To actually filter by a nested/reference field via the DSL, see **"Escape hatches"** right after this block.

```java
// OPTION 1: Type-safe chained access — produces the path string "homeAddress.city"
var employees = db.select()
    .fromType("Employee")
    .where()
        .property($homeAddress.__($city).__).eq().value("San Francisco")
    .documents();

// Benefits: compile-time verification, type checking, refactoring safety, IDE autocomplete

// OPTION 2: String-based (FALLBACK)
var employees2 = db.select()
    .fromType("Employee")
    .where()
        .property("homeAddress.city").eq().value("San Francisco")
    .documents();

// Use strings when: chaining becomes complex, dynamic field names, prefer simplicity

// Deep nesting (3+ levels):
// Person → Address → Location → country
var result = db.select()
    .fromType("Person")
    .where()
        .property($address.___($location).__($country).__).eq().value("USA")
    .documents();
//             ^^^               ^^
//             3 underscores     2 underscores (terminal)
//             DataField chain   Field terminal

// Chaining rules:
// - ___() (3 underscores): Chain DataField → DataField (continue chaining)
// - __() (2 underscores): Chain DataField → Field (terminal, use .__ for name)
// - Final .__ is alias for .name()

// Type-safe updates (both direct value and Supplier supported)
Update.use(db)
    .select("Employee")
    .whereEq($employeeId.__, "EMP001")
    .upsert()
    .__($homeAddress.__($city).__, "Palo Alto")  // Direct value (simple)
    .__($age, () -> calculateAge())              // Supplier (error isolation)
    .saveDocument();

// String-based (both direct value and Supplier supported, no type checking)
Update.use(db)
    .select("Employee")
    .whereEq("employeeId", "EMP001")
    .upsert()
    .__("homeAddress.city", "Palo Alto")         // Direct value (simple)
    .__("age", () -> calculateAge())             // Supplier (error isolation)
    .saveDocument();

// Why Supplier? Error isolation - if one field fails, others still update
Update.use(db)
    .select("Employee")
    .whereEq($employeeId.__, "EMP001")
    .upsert()
    .__($name, "Alice")                          // ✓ Updates
    .__($age, () -> riskyCalculation())          // ✗ Throws exception (isolated)
    .__($email, "alice@example.com")             // ✓ Still updates
    .saveDocument();
// Without Supplier, first exception would abort entire upsert
```

**Escape hatches — how to actually filter by a nested or reference field:**

The native DSL WHERE can't do it (above). Reading nested/reference values *out of a fetched record* is fully supported and type-safe; only **filtering by them inside `.where()`** is the gap. Four working approaches, cheapest-to-set-up first:

**1. Post-filter in Java (the general answer — works for embedded AND references).** Prefilter on a flat, indexed field to keep the candidate set small, then filter the nested/reference part on the deserialized object:
```java
// Embedded: index-accelerated flat prefilter, then nested filter in Java (nested part is NOT index-accelerated)
var inSF = db.select().fromType(Employee.__)
    .where().property($department.__).eq().value("Eng")   // flat + indexed
    .documents().stream()
    .map(Employee::of)
    .filter(e -> "San Francisco".equals(e.homeAddress().city()))
    .toList();

// Reference/LINK: resolve then filter (one extra fetch per row — prefilter hard first)
var acmeInvoices = db.select().fromType(Invoice.__)
    .where().property($status.__).eq().value("OPEN")
    .documents().stream()
    .map(Invoice::of)
    .filter(inv -> "Acme".equals(inv.customer().resolve(db).name()))
    .toList();
```

**2. Graph traversal (the one "nested" thing the DSL does natively — edges only).** If the relationship is an EDGE, walk it instead of filtering by it — this is index/adjacency-backed, not a scan:
```java
var alicesColleagues = db.select().fromType(Person.__)
    .where().property($email.__).eq().value("alice@example.com")   // flat entry point
    .vertices().outVertices(Knows.__).toList();
```
(Does not apply to embedded documents or document `LINK` fields — only true graph edges.)

**3. Denormalize (the only way to get an *indexed* nested filter through the DSL).** Copy the needed sub-field up to the parent as a flat, indexed property, then the DSL filters it natively and fast:
```java
@ArcadeData
public final class Employee extends Employee_A {
    String employeeId;
    Address homeAddress;
    String  homeCity;   // denormalized copy of homeAddress.city — keep in sync on write

    public static final TypeDef<Employee> TYPEDEF =
        schemaBuilder().factory(Employee::new).unique($employeeId).lsmIndex($homeCity).__();
}
// Now this is a native, index-backed DSL filter:
db.select().fromType(Employee.__).where().property($homeCity.__).eq().value("San Francisco").documents();
```

**4. SQL for that one query (the one-liner, index-capable).** When the filter is genuinely nested/reference-shaped and you don't want to denormalize, use SQL — its evaluator resolves dot-paths and link hops and can use a dotted-path index:
```java
try (var rs = db.query("sql", "SELECT FROM Employee WHERE homeAddress.city = ?", "San Francisco")) {
    while (rs.hasNext()) { var e = Employee.of(rs.next().toElement().asDocument()); /* ... */ }
}
```

**Rule of thumb:** hot path + you control the schema → **denormalize (3)**; occasional query → **SQL (4)**; relationship is a real edge → **traversal (2)**; everything else / quick-and-dirty → **post-filter in Java (1)**.

**Architecture:**
- Static FIELDS references passed through annotation processor
- Zero reflection (no `Class.forName()`, no `getDeclaredFields()`)
- Automatic dependency-ordered registration
- Type-safe nested field access
- Automatic serialization to ImmutableEmbeddedDocument
- Automatic deserialization via `fromArcadeDocument()`
- Works for both single objects and List<T>

---

### PRP-22: Graph Types (VERTEX, EDGE, DOCUMENT)

**Feature:** @ArcadeData now supports graph types via `type` parameter.

**Examples:**

```java
// DOCUMENT (default)
@ArcadeData
public final class Order extends Order_A {
    String orderId;
    Double amount;

    public static final TypeDef<Order> TYPEDEF =
        schemaBuilder().factory(Order::new).unique($orderId).__();
}

// VERTEX for graph nodes
import static datapotter.arcadedbhelper.ArcadeType.*; // prefer static import, anyway u need to import and u need only the constants not the enum 
@ArcadeData(type = VERTEX)
public final class Person extends Person_A {
    String name;
    String email;

    public static final TypeDef<Person> TYPEDEF =
        schemaBuilder().factory(Person::new).unique($email).__();
}

// EDGE for graph relationships
import static datapotter.arcadedbhelper.ArcadeType.*; // prefer static wildcard import 
@ArcadeData(type = EDGE)
public final class Knows extends Knows_A {
    String since;
    String strength;

    // This is optional, in schema registration we can directly call Knows.schemaBuilder().__() 
	// versus Knows.TYPEDEF 
	// until there is an unique or some index .... there is no sence of making this 
	//public static final TypeDef<Knows> TYPEDEF = schemaBuilder().__();
}
```

**Generated schema builder:**
```java
// In Person_A.java
public static SchemaBuilder<Person> schemaBuilder() {
    return defType(Person.class, ArcadeType.VERTEX).fields(FIELDS);
}
```

**Schema initialization:**
```java
// Creates VertexType instead of DocumentType
InitDoc.initDocTypes(db, Person.TYPEDEF, Knows.TYPEDEF);
```

**Runtime graph operations (use ArcadeDB native API):**
```java
db.transaction(() -> {
    // Create vertices
    MutableVertex bob = db.newVertex("Person");
    bob.set("name", "Bob");
    bob.set("email", "bob@example.com");
    bob.save();

    MutableVertex charlie = db.newVertex("Person");
    charlie.set("name", "Charlie");
    charlie.set("email", "charlie@example.com");
    charlie.save();

    // Create edge
    MutableEdge knowsEdge = bob.newEdge("Knows", charlie, true);
    knowsEdge.set("since", "2020");
    knowsEdge.set("strength", "strong");
    knowsEdge.save();

    // Traverse
    Iterable<Vertex> friends = bob.getVertices(Vertex.DIRECTION.OUT, "Knows");
    for (Vertex friend : friends) {
        System.out.println("Friend: " + friend.getString("name"));
    }
});
```

**Design philosophy:**
- @ArcadeData handles **schema declaration** (type, fields, indexes)
- ArcadeDB API handles **runtime operations** (graph traversal, edge creation)
- No magic abstraction - clear boundaries

---

### PRP-04: Transient Reference Types (LINK) & Graph Traversal

> **Supersedes** the old "PRP-22 Phase 3" manual `addLinkType` / `$$customerRef` LINK approach. That bolt-on (schema-only, no Java field, raw `orderDoc.set("customerRef", rid)`) is **retired**. References are now first-class, type-driven, and reflection-free.

**First principle:** every `@ArcadeData` object is a detached, in-memory **snapshot** — there is no live DB session behind it. So a reference can never lazy-fault from a getter. A DTO-typed field **embeds a copy**; a field wrapped in `Link<T>` stores only a **reference** (the target's RID) and is resolved separately, explicitly, never by hidden I/O.

```java
@ArcadeData
public final class Invoice extends Invoice_A {
    String invoiceId;
    Double amount;

    Link<Customer>          customer;   // 1:1  → schema LINK
    LinkList<LineItem>      items;      // 1:n  → LIST of links
    LinkMap<String,Account> ledgers;    // n:n  → MAP of links

    Address billingAddress;             // a DTO field still embeds by value (PRP-21) — unchanged

    public static final TypeDef<Invoice> TYPEDEF =
        schemaBuilder().factory(Invoice::new).unique($invoiceId).__();
}
```

The single distinction is self-documenting: **`Customer c` = embed a copy; `Link<Customer> c` = reference a record.** No annotation, no `$$` convention — it falls out of the existing type-driven model. Link targets are auto-registered as schema dependencies, exactly like embedded types.

#### The one rule: `db` means fetch

A method that takes a `Database` performs the (single, explicit) I/O and returns resolved objects. A method without `db` is a pure carrier and never touches the database.

```java
inv.customer();                  // Link<Customer> — carrier, no I/O
inv.customer().rid();            // "#12:3"        — the identity (always present when set)
inv.customer().isResolved();     // did the query project the target?
inv.customer().get();            // Customer       — the projected target, or throws UnresolvedLinkException
inv.customer().resolve(db);      // Customer       — the explicit fetch
inv.items().resolveAll(db);      // List<LineItem> — batch fetch
```

**Zero reflection.** Resolution instantiates targets through a generated `Customer::new` factory the carrier captured at codegen time — never `Class.newInstance()`. So links are GraalVM-native-safe like the rest of DataHelper. (Internally `Link<T>` holds a `Supplier<T>`, not a `Class<T>`.)

#### Setting links — fluent, by object / rid / collection

```java
new Invoice().invoiceId("INV1").amount(99.9)
    .customer(cust)                  // by object  → Link.of(cust), reads cust.$rid()
    .customer("#12:3")               // by rid     → Link.ofRid(...)
    .items(List.of(a, b))            // List<Target>  → LinkList
    .ledgers(Map.of("main", acc))    // Map<K,Target> → LinkMap
    .in(db).whereEq($invoiceId, "INV1").upsert();
```

`customer(Customer)` requires the target to be persisted (to have a `$rid()`). The base `customer(Link<Customer>)`, `items(LinkList<LineItem>)`, `ledgers(LinkMap<...>)` setters also exist.

#### Query-shaped resolution — let the DB do the join

A nested projection comes back **already resolved** — the preferred path. Read a query row with `fromResult(Result)`:

```java
try (var rs = db.query("sql", "SELECT *, customer:{*} FROM Invoice WHERE invoiceId = ?", "INV1")) {
    Invoice inv = new Invoice().fromResult(rs.next());
    inv.customer().get().name();     // resolved straight from the row — no extra lookup, no I/O
}
```

`SELECT customer` (bare) → an identity-only `Link` (`isResolved() == false`); `SELECT customer:{...}` → a resolved `Link` shaped by exactly the projected fields. The query decides what's resolved, never a hidden getter.

#### Self-identity, edge endpoints, vertex adjacency

Every loaded record carries its own read-only `$rid()`. Graph records (`@ArcadeData(type = VERTEX | EDGE)`) extend the same carrier vocabulary:

```java
loaded.$rid();                              // "#12:3" — read-only, bound by the load layer

// EDGE — exactly one source + one target endpoint, so each is a single Link
//   @ArcadeData(type = EDGE) public final class Knows extends Knows_A { String since; }
LinkKnows knows = LinkKnows.of(edgeDoc);    // of(Document) captures @out/@in on load
knows.$out();                               // Link<?> — source-vertex reference (RID present)
knows.$in();                                // Link<?> — target-vertex reference
knows.$out().resolve(db, Person::new);      // Person  — fetch it (supply the type as a factory)

// VERTEX — adjacency over edge types is a LinkList (many neighbours), resolved
bob.$out (db, Person::new, Knows.class);    // LinkList<Person> — out-neighbours
bob.$in  (db, Person::new, Knows.class);    // in-neighbours
bob.$both(db, Person::new, Knows.class);    // both directions
```

- **Edge endpoints are untyped `Link<?>`** (a typed `@ArcadeData(out=…, in=…)` declaration is a future refinement), so resolve them by supplying the type: `resolve(db, Person::new)`.
- **Edge types are passed as generated type classes** (`Knows.class`), not strings — refactor-safe; the engine label is the class's simple name. No `edgeTypes` means "all edge types".
- **Creating vertices:** from datahelper 1.1 the instance DSL and `Upsert` both work on a `@ArcadeData(type = VERTEX)` class (`new Person().email("a@x").in(db).whereEq("email","a@x").upsert()`), so the native `db.newVertex(Person.__)` is now a choice rather than the only route. **Creating edges** still uses the native API (`bob.newEdge(Knows.__, charlie)`); a typed edge-creation DSL is deferred, and a keyed upsert on an EDGE type is refused by design.
- ⚠️ **On ArcadeDB 26.x these traversal helpers require datahelper 1.1 or later.** On 1.0 every one of them throws `NoSuchMethodError: RID.<init>(BasicDatabase, String)` — as does `Link.resolve(db)` and the write path for every `Link` field. See [ArcadeDB 26.x: what broke and what to change](#arcadedb-26x-what-broke-and-what-to-change).
- **These address by `$rid()`**, so they fetch the vertex behind the snapshot. If you already hold the `Vertex` — which is what an indexed lookup gives you — use [`Traverse`](#7-vertex-addressed-adjacency-traverse) instead and skip the round-trip. Same traversal, same results; over a subtree walk it saves one redundant record fetch per node.

#### Reserved field names

`$rid()` / `$out()` / `$in()` are generated accessors; their backing storage is `__rid` / `__out` / `__in`, so a data field literally named `rid` / `out` / `in` still compiles (its `$rid`/`$out`/`$in` *symbol* coexists with the accessor) — but the processor warns, because the spelling is confusing. Prefer other names.

#### The unified RID algebra

Every reference ArcadeDB has is a RID or a list of RIDs, so three carriers model all of it:

| ArcadeDB reference | Carrier | Accessor |
|---|---|---|
| record's own identity | scalar | `$rid()` (read-only) |
| document `LINK` | `Link<T>` | `customer()` |
| `LIST`/`MAP` of links | `LinkList<T>` / `LinkMap<K,T>` | `items()` / `ledgers()` |
| edge `@out` / `@in` | `Link<?>` each | `$out()` / `$in()` |
| vertex adjacency (by edge type) | `LinkList<T>` | `$out/$in/$both(db, T::new, Edge.class)` |

---

*Last Updated: 2026-07-31*
*Based on: ArcadeDB 26.7.3, datapotter-arcadedbhelper 1.7 (annotation processor 1.3), Gson 2.14.0*
*Revision (2026-07-31): moved the baseline from ArcadeDB 25.7.1 / datahelper 1.0 to 26.7.3 / 1.3, and added [ArcadeDB 26.x: what broke and what to change](#arcadedb-26x-what-broke-and-what-to-change) — the removed `RID(BasicDatabase, …)` constructors (`RID.create(...)` now), which had silently disabled every `Link<T>` resolve, every `Link` write and all of `$out`/`$in`/`$both` on 26.x; VERTEX record creation dispatching to `newVertex` instead of `newDocument`; and the Gson 2.11 floor that makes 26.x die at startup on an older Gson while the build stays green. Added §6 [Typed reads](#6-typed-reads-query), §7 [Vertex-addressed adjacency](#7-vertex-addressed-adjacency-traverse) and §8 [Typed field-level upsert](#8-typed-field-level-upsert-upsert) for the 1.2/1.3 DSLs — `Query` closes the last hole in the typed API, since `.__` yields a bare `String` that binds no value type, and it refuses a nested-field filter outright rather than passing the engine's silent empty result through. All claims verified by running the datahelper suite against 26.7.3 (9 test classes, 2 of them new: `UpsertIntegrationTest` for `Upsert` and vertex-aware creation, `QueryIntegrationTest` for `Query`/`Traverse` — none of which had any coverage before).*
*New: PRP-04 (Transient Reference Types — `Link`/`LinkList`/`LinkMap`, query-shaped resolution, edge endpoints, vertex adjacency; reflection-free `T::new` resolution). Earlier: PRP-21 (Nested Types), PRP-22 (Graph Types).*
*Note: the PRP-22 Phase-3 manual `addLinkType`/`$$` LINK approach is retired in favour of the `Link<T>` carriers documented above.*
*Docs revision (2026-07-01): reworked the query sections to steer AI agents toward the native `db.select()` DSL instead of SQL strings — added the "DSL FIRST" directive + rationale (§2), a complete DSL reference covering `and`/`or` precedence, `in`/`between`/`isNull`/`isNotNull`, `count`/`exists`/`stream`, `SelectIterator` methods, graph traversal, and compiled/parameterized reuse (§3), and worked examples for compound/range/in/null conditions, counts, and traversal (§4). Verified against engine sources (`com.arcadedb.query.select.*`) and the datahelper test suite. Corrected the PRP-21 claim that nested embedded-field filtering works via the native DSL — it does not, and fails *silently* (returns empty): the executor re-evaluates every candidate with a flat `record.get()`, so even a dotted-path index doesn't help (it finds rows, then the flat re-filter discards them). Added an "Escape hatches" subsection with the four working alternatives — post-filter in Java, graph traversal (edges), denormalize a flat indexed field, or SQL. Verified against `SelectExecutor`/`SelectPropertyValue`/`Immutable|MutableDocument.get`.*
