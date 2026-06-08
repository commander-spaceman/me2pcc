# Project Map

**Purpose:** Go library for reading Mass Effect 2 OT PCC (package) files. Single-package static library with no CLI; exports a `pcc` API consumed by other tools in the [`pcc-toolkit`](https://github.com/commander-spaceman/pcc-toolkit) ecosystem.

## Notes for AI Agents

- **Entry points:** `reader.go` — all public read functions (`ReadFile`, `ReadFileRaw`, `ReadFileFromBytes`, etc.) are the main API surface. `types.go` defines the core data structures returned by every read.
- **Main patterns:** Single Go package (`pcc`). All files share one namespace. Public functions are upper-cased; internal helpers are lower-cased. Little-endian binary parsing throughout. No interfaces, no concurrency — procedural data extraction and decoding.
- **General rule:** Read this file before proposing structural changes or modifying multiple modules.

---

## 1. Types & Constants

Core data structures, constants, and enums shared by the entire package.

```text
types.go
```

**Main responsibilities:**

- Define PCC binary constants (magic numbers, flags, header/chunk sizes)
- Define `Header`, `Import`, `Export`, `FileSummary`, `ExportDetail`, `ContainerHit` structs
- Define `GameProfile` enum (`me2_ot`, `le2`, `me3_ot`, `le3`, `unknown`) and `InferGameProfile`
- `FileSummary.RequireME2()` guard for ME2-OT-specific code paths

**Key files:**

- `types.go`: All types and constants used across the library.

**Relationships:**

- Consumed by every other file. No internal dependencies.

---

## 2. File Reading & Parsing

Public API surface and low-level PCC binary-parsing internals.

```text
reader.go
```

**Main responsibilities:**

- Public entry points: `ReadFile`, `ReadFileHeaderOnly`, `ReadFileRaw`, `ReadFileExports`, `ReadFileFromBytes`, `ReadFileFromBytesAndExports`
- Internal parsers: `parsePccHeader`, `parsePccNames`, `parsePccImports`, `parsePccExports`
- Name resolution: `resolveExportNames` populates `ObjectName` and `ClassName` on each export

**Key files:**

- `reader.go`: Entire file-reading pipeline — read → decompress (if needed) → parse header/names/imports/exports → resolve names → return `FileSummary`.

**Relationships:**

- Calls `decompress.go` for compressed files
- Calls `strings.go` for `readI32`, `readU32`, `readUnrealString`, `resolveName`

---

## 3. Decompression

LZO chunk-based decompression for compressed PCC files (ME2 OT format).

```text
decompress.go
```

**Main responsibilities:**

- `decompressME2OT`: locate compression info, parse chunk tables, decompress LZO blocks, reassemble decompressed file
- `locateCompressionInfoOffsetME2OT`: navigate past header/folder/generations to find compression metadata

**Key files:**

- `decompress.go`: All decompression logic.

**Relationships:**

- Depends on external library `github.com/commander-spaceman/me2lzo/decompress` for LZO block decompression
- Called by `reader.go` when `CompressedFlag` is set in the header

---

## 4. String & Binary I/O Utilities

Low-level helpers for reading integers and Unreal Engine strings from raw byte slices.

```text
strings.go
```

**Main responsibilities:**

- `readI32` / `ReadRawI32`: little-endian int32 from byte slice
- `readU32`: little-endian uint32 from byte slice
- `readUnrealString`: parse Unreal's length-prefixed string format (ASCII or UTF-16)
- `resolveName` / `ResolveName`: index-to-string lookup against the name table

**Key files:**

- `strings.go`: All primitive I/O operations.

**Relationships:**

- Used by every parser file (`reader.go`, `properties.go`, `unreal_props.go`, `decompress.go`)

---

## 5. Property Tag Parsing & Array Analysis

Low-level property tag scanning, heuristics for identifying property boundaries, and array layout analysis.

```text
properties.go
```

**Main responsibilities:**

- `ParsePropertyTags`: scan export serial data for `(name, type, size, index)` tuples
- Heuristic metadata resolution: `skipTagMeta`, `resolveBoolOrArrayMeta`, `isPlausibleTagStart`
- Array helpers: `ReadArrayPropertyCount`, `ReadArrayPropertyI32Values`, `AnalyzeArrayPropertyLayout`, `ReadArrayPropertyStructI32Matrix`, `ReadArrayPropertyStructHeadI32`
- Conversation keys: `ExtractBioConversationKeyProperties` — find `EntryList`/`ReplyList`/`SpeakerList`/`StartingList` arrays inside BioConversation exports
- Name-based scanning: `FindInt32ArrayByName`, `FindIntPropertyByName`, `FindStructArrayItemStarts`
- Batch computation: `ComputeExportProperties` — compute tags and/or semantic properties for all exports

**Key files:**

- `properties.go`: The largest file — property tag parsing, array analysis, and conversation-specific extraction.

**Relationships:**

- Uses `strings.go` for `readI32`, `resolveName`
- Uses `types.go` for `PropertyTag`, `ArrayLayoutInfo`, `PropertyTypeNames`, `ExportProperties`
- Complements `unreal_props.go` (tags vs. decoded values)

---

## 6. Semantic Property Decoding

Full property-value decoding into typed Go values.

```text
unreal_props.go
```

**Main responsibilities:**

- `ParsePropertyCollection`: parse an export's property tags and decode their values into `map[string]ParsedProperty`
- `ParseStructArrayItemsAsPropertyCollections`: decode arrays of `StructProperty` items using four progressively aggressive strategies (repeated-tag matching, fixed striding, plausible-start scanning, sequential fallback)
- `parsePropertyHeader` / `readFName`: lower-level header parsing reused across the codebase
- `HasStructSignature`: check if a byte range looks like a valid struct property start

**Key files:**

- `unreal_props.go`: Semantic value decoding — complements `properties.go` which focuses on tag-level structure.

**Relationships:**

- Uses `strings.go` for `readI32`, `resolveName`
- Uses `types.go` for `ParsedProperty`, `PropertyTypeNames`
- Called by `properties.go` (`ComputeExportProperties` with `includeSemantic=true`)

---

## 7. Container Mapping

Map raw byte offsets (e.g., string references) to the export that contains them.

```text
containers.go
```

**Main responsibilities:**

- `MapOffsetsToContainers`: given a map of `strRef → []absoluteOffset` and the export list, find which export owns each offset
- `bestContainingExport`: internal heuristic that prefers the smallest named-class export containing the offset

**Key files:**

- `containers.go`: Small utility with two functions.

**Relationships:**

- Uses `types.go` for `Export`, `ContainerHit`
- Used by downstream tools (not internally within this package)

---

## 8. Tests

Unit tests using hand-crafted binary PCC fixtures.

```text
reader_test.go
properties_test.go
```

**Main responsibilities:**

- `reader_test.go`: Tests header parsing, name/import/export table parsing, name resolution, export filtering, game profile inference, string decoding, and container mapping — all against `buildMinimalPCC` fixtures.
- `properties_test.go`: Tests `FindInt32ArrayByName` and `FindIntPropertyByName` using manually assembled byte buffers.

**Key files:**

- `reader_test.go`: Core test suite (408 lines); covers the reading pipeline end-to-end.
- `properties_test.go`: Property scanning tests (71 lines).

**Relationships:**

- Both files are in the `pcc` package (white-box tests).

---

## 9. Build & Configuration

```text
go.mod
go.sum
.gitignore
LICENSE
```

- `go.mod`: Module `github.com/commander-spaceman/me2pcc`, Go 1.25.5, single external dependency on `github.com/commander-spaceman/me2lzo`
- `LICENSE`: MIT
- `.gitignore`: Standard Go ignores (binaries, test artifacts, IDE files)
