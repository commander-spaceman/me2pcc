# me2pcc

Go library for reading Mass Effect 2 OT PCC (package) files.

## Installation

```bash
go get github.com/commander-spaceman/me2pcc
```

## Usage

```go
import pcc "github.com/commander-spaceman/me2pcc"

// Read a PCC file fully
summary, err := pcc.ReadFile("BioD_CitHub.pcc")

// Read with raw decompressed bytes
rawData, summary, err := pcc.ReadFileRaw("BioD_CitHub.pcc")

// Parse properties from an export
props, _ := pcc.ParsePropertyCollection(rawData, summary.Names, export.SerialOffset, export.SerialSize)
```

## API

### File reading

| Function                                             | Description                               |
| ---------------------------------------------------- | ----------------------------------------- |
| `ReadFile(path)`                                     | Read and parse a PCC file                 |
| `ReadFileHeaderOnly(path)`                           | Parse only the header and game profile    |
| `ReadFileRaw(path)`                                  | Read, decompress, parse, return raw bytes |
| `ReadFileExports(path, predicate)`                   | Read and filter exports by predicate      |
| `ReadFileFromBytes(data, path)`                      | Parse from in-memory bytes                |
| `ReadFileFromBytesAndExports(data, path, predicate)` | Parse and filter exports from bytes       |

### Property parsing

| Function                                                    | Description                                      |
| ----------------------------------------------------------- | ------------------------------------------------ |
| `ParsePropertyTags(data, names, offset, size, strict)`      | Parse property name/type/size tags               |
| `ParsePropertyCollection(data, names, offset, size)`        | Parse property values into a map                 |
| `ParseStructArrayItemsAsPropertyCollections(...)`           | Parse array of structs into property collections |
| `ComputeExportProperties(rawData, summary, tags, semantic)` | Compute properties for all exports               |
| `ExtractBioConversationKeyProperties(...)`                  | Extract BioConversation key array properties     |

### Utility

| Function                                            | Description                             |
| --------------------------------------------------- | --------------------------------------- |
| `ReadRawI32(data, offset)`                          | Read a little-endian int32              |
| `ResolveName(data, offset, names)`                  | Resolve a name table index              |
| `MapOffsetsToContainers(exports, offsets, dataLen)` | Map byte offsets to containing exports  |
| `InferGameProfile(uv, lv)`                          | Infer game profile from version numbers |

## Types

| Type             | Description                                          |
| ---------------- | ---------------------------------------------------- |
| `FileSummary`    | Parsed PCC file with header, names, imports, exports |
| `Header`         | PCC header fields                                    |
| `Export`         | Export table entry                                   |
| `Import`         | Import table entry                                   |
| `PropertyTag`    | Parsed property tag (name, type, size, offset)       |
| `ParsedProperty` | Decoded property value                               |
| `ContainerHit`   | Byte offset mapped to owning export                  |
| `GameProfile`    | Game profile enum (me2_ot, le2, me3_ot, le3)         |

## License

MIT
