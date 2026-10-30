# Migration guide — DataHelper / ArcadeDB Helper moved to the `datapotter` org (2026-09-11)

`xyz-jphil/datahelper` (one GitHub repo holding both libraries plus experiments) has been split into three repos under the new `datapotter` GitHub org, and renamed from the `xyz-jphil`/`xyz.jphil` namespace to `datapotter`. The API, behaviour and version numbers are unchanged — this is a rename and a repo split, not a rewrite. The old repo `xyz-jphil/datahelper` is being retired in favour of the repos below.

## Where things live now

- `datapotter/datahelper` (public) — the core library: `annotations`, `base`, `processor`, `json`. The old `teavm/` module was dropped (it was an empty stub with no classes).
- `datapotter/arcadedbhelper` (public) — the ArcadeDB integration, was `xyz-jphil-arcadedb-datahelper` + `xyz-jphil-arcadedb-datahelper-processor`. The two module folders are now named `core/` and `processor/`.
- `datapotter/experiments` (private) — the performance/experiment harnesses (`test-performance-*`, `test-teavm`), moved as-is. Not relevant to library consumers.

Both public repos ship an Apache License 2.0 (`LICENSE`), which the old repo never had.

## Coordinate changes

groupId: `io.github.xyz-jphil` → `io.github.datapotter`, everywhere.

DataHelper core (`datapotter/datahelper`), same version numbers as before:

- `xyz-jphil-datahelper-annotations` → `datapotter-datahelper-annotations`
- `xyz-jphil-datahelper-base` → `datapotter-datahelper-base`
- `xyz-jphil-datahelper-processor` → `datapotter-datahelper-processor`
- `xyz-jphil-datahelper-json` → `datapotter-datahelper-json`

ArcadeDB helper (`datapotter/arcadedbhelper`), same version numbers as before:

- `xyz-jphil-arcadedb-datahelper` → `datapotter-arcadedbhelper`
- `xyz-jphil-arcadedb-datahelper-processor` → `datapotter-arcadedbhelper-processor`

## Package changes

- `xyz.jphil.datahelper` → `datapotter.datahelper` (and its subpackages, e.g. `xyz.jphil.datahelper.json` → `datapotter.datahelper.json`, `xyz.jphil.datahelper.processor.util` → `datapotter.datahelper.processor.util`)
- `xyz.jphil.arcadedb.datahelper` → `datapotter.arcadedbhelper` (and `xyz.jphil.arcadedb.datahelper.processor` → `datapotter.arcadedbhelper.processor`)

Every class keeps its simple name — only the package prefix changes. `@Data`, `@DataHelper`, `@ArcadeData`, `DataHelper_I`, `ArcadeDoc_I`, `Link`/`LinkList`/`LinkMap`, `Query`, `Upsert`, `Traverse`, `SchemaBuilder`, `TypeDef` etc. all work exactly as documented in the tutorial — just import them from the new package.

## What a consumer project needs to change

1. Bump the `groupId` on every DataHelper/ArcadeDB-helper dependency to `io.github.datapotter`, and the `artifactId` per the mapping above (version numbers stay the same for now).
2. Update the `annotationProcessorPaths` entries the same way.
3. Find/replace imports: `xyz.jphil.datahelper` → `datapotter.datahelper`, `xyz.jphil.arcadedb.datahelper` → `datapotter.arcadedbhelper`.
4. Re-run `mvn clean install` — the annotation processors are not yet on Maven Central, so for now these still need to be built from source (`git clone` + `mvn install` per module, in dependency order: `annotations` → `base` → `processor` → `json` for datahelper, then `core` → `processor` for arcadedbhelper) rather than resolved from a public repository. A proper Maven Central release is planned separately; this guide will be updated once that lands and the groupId/artifactId above become fetchable directly.

## Before/after example

Old:
```xml
<dependency>
    <groupId>io.github.xyz-jphil</groupId>
    <artifactId>xyz-jphil-arcadedb-datahelper</artifactId>
    <version>1.7</version>
</dependency>
```
```java
import xyz.jphil.arcadedb.datahelper.ArcadeData;
```

New:
```xml
<dependency>
    <groupId>io.github.datapotter</groupId>
    <artifactId>datapotter-arcadedbhelper</artifactId>
    <version>1.7</version>
</dependency>
```
```java
import datapotter.arcadedbhelper.ArcadeData;
```
