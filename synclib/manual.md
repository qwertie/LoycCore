---
title: "SyncLib Reference Manual"
tagline: "One synchronizer method to rule them all — ultra flexible with no DTOs"
layout: article
toc: true
date: 18 Jul 2026
---

SyncLib is a .NET serialization system for real-world projects where traditional attribute-based serialization falls flat, forcing you to use DTOs.

Attribute-based libraries don't allow the level of control you need for

- Type conversion during serialization (Color values <=> strings, and multimaps <=> Lists)
- Nonlocal representation changes (Start + End Date <=> Start Date + Duration)
- Multiple serialization formats for backward compatibility purposes (v1/v2)
- Serializing the same type in multiple ways (storing Color as string in some places and as integer in others)

With SyncLib you can support many serializers at once in about half the lines of code as DTO-based transformations, while getting great performance.

To use it, write one *synchronizer* method per type — a method that reads **or** writes each field depending on the mode. That single method drives loading, saving, and even schema generation, in every supported data format. (See the [home page](/index.html) for a side-by-side comparison with a traditional serializer.)

The library ships in three assemblies:

| Assembly | Namespace | Contents |
|---|---|---|
| Loyc.Essentials.dll | `Loyc.SyncLib` | The core interfaces and helpers, plus the **SyncBinary** format |
| Loyc.SyncLib.SyncJson.dll | `Loyc.SyncLib` | The **SyncJson** format (JSON + JSON Schema) |
| Loyc.SyncLib.SyncProtobuf.dll | `Loyc.SyncLib` | The **SyncProtobuf** format (Protocol Buffers + .proto schema) |

Low-level building blocks live in `Loyc.SyncLib.Impl` (summarized [at the end](#the-implementation-layer-loycsynclibimpl)). This page is a summary of the whole library; detailed API documentation will be generated from the source code separately.

## The synchronizer concept

A synchronizer is a method (or lambda) matching this delegate:

~~~csharp
public delegate T SyncObjectFunc<in SyncManager, T>(SyncManager sync, T? value);
~~~

It is called with an object to save (when writing) or a default/null `value` (when reading), and it must call `sync.Sync(...)` once per field. Each `Sync` call *returns* the field's value — the saved value when writing, the loaded value when reading — so the same code works in both directions:

~~~csharp
class Person
{
    public string? Name;
    public int Age;
    public Person[]? Siblings;
}

public static Person SyncPerson<SM>(SM sm, Person? obj) where SM : ISyncManager
{
    sm.CurrentObject = obj ??= new Person();  // see "Deduplication and cycles"
    obj.Name     = sm.Sync("Name", obj.Name);
    obj.Age      = sm.Sync("Age", obj.Age);
    obj.Siblings = sm.SyncList("Siblings", obj.Siblings, SyncPerson);
    return obj;
}

// Usage: the same function serializes and deserializes, in any format
string json  = SyncJson.WriteString(jack, SyncPerson, options);
Person? j1   = SyncJson.Read<Person>(json, SyncPerson);
var bytes    = SyncBinary.Write(jack, SyncPerson);
Person? j2   = SyncBinary.Read<Person>(bytes.ToArray(), SyncPerson);
~~~

**Rules of the contract:**

- When writing, every `Sync` method returns the same value passed in, so the assignments are harmless.
- When reading, your function must construct the object (`obj ??= new Person()`) and store what `Sync` returns.
- Fields must be synchronized unconditionally and (for formats without field names) in a fixed order. If the set of fields must change, use explicit versioning: write a version number field first, then branch on it. This works in every format and is the intended substitute for attribute-driven schema evolution.
- If your code must behave differently when loading vs. saving, test `sm.IsReading` / `sm.IsWriting` rather than comparing `Mode` directly — these properties classify the Schema, Query and Merge modes correctly.

**Generic vs. interface-typed synchronizers.** Each format exposes its manager as a *struct* (e.g. `SyncJson.Writer`). Writing your synchronizer as a generic method (`SyncPerson<SM> where SM : ISyncManager`, as above) lets the JIT specialize it per format — no boxing or virtual dispatch. Alternatively, write it against plain `ISyncManager` and use the `I`-suffixed helpers (`WriteI`, `ReadI`, `WriteStringI`, `WriteSchemaI`): simpler, works with every format at once, but slower. A third option for maximum speed is a struct implementing `ISyncObject<SM,T>`, usable with the same helper methods.

## The ISyncManager interface

All formats implement `ISyncManager`. Its members fall into a few groups.

### Modes

`Mode` returns a `SyncMode`: **Reading** (loading data), **Writing** (saving), **Schema** (no data flows; the synchronizer is executed once so the manager can learn the shape of the type), **Query** (a saving variant that may skip parts of the graph), and **Merge** (a bidirectional synchronization mode — *reserved; not supported by any current implementation*). `IsReading` is true in Reading/Schema/Merge; `IsWriting` is true in Writing/Query/Merge.

### Field identity: FieldId

Every `Sync` method takes a `FieldId`, a struct holding a `Name` (string) and an `Id` (int). A plain string converts implicitly, `("Name", 5)` tuples supply an explicit ID, and a `Symbol` from a private `SymbolPool` can supply both. Formats that use field names (JSON) use `Name`; formats that use numbers (protobuf) use `Id`, assigning 1, 2, 3… automatically when no ID is given (`NeedsIntegerIds` tells you which kind you're talking to). SyncBinary ignores names entirely. Pass `null` to mean "the next field, whatever it's called".

### Synchronizing primitives

`Sync(FieldId, T)` overloads exist for `bool`, all integer types, `float`, `double`, `decimal`, `BigInteger`, `char`, and `string`, plus a nullable variant of each. There are also bitfield overloads — `Sync(name, value, int bits, bool signed = true)` for `int`/`long`/`BigInteger` — which store a fixed number of bits. `SyncRef(name, ref field)` variants assign the result back to a `ref` parameter for you. Readers are encouraged to support widening conversions (bool→byte, byte→int, int→float, float→string, char→string).

Helpers for common non-primitive values (all format-independent):

- `SyncEnumAsString(name, e)` — enum as its name string. (Enums are otherwise stored numerically.)
- `SyncDateAsString(name, dt, preferredFormat?, parseMode?)` — ISO-8601 by default; `SyncDateAsDayNumber(name, dt, asInt32)` — OLE Automation day number.
- `SyncTimeAsString`, `SyncTimeAsSeconds`, `SyncTimeAsMinutes`, `SyncTimeAsDays` — `TimeSpan` in your unit of choice.

### Sub-objects and ObjectMode

To synchronize a nested object, pass its synchronizer: `sm.Sync("Border", obj.Border, SyncShape, mode)`. The `ObjectMode` flags control how the child is treated:

| Flag | Meaning |
|---|---|
| `Normal` | An object with named fields (default) |
| `List` | Variable number of unnamed items |
| `Tuple` | Fixed number of unnamed items; the length is *not* stored, so the reader must pass the same `tupleLength` |
| `Deduplicate` | Write the object once and back-reference it afterward; enables cyclic graphs. **Default for sub-objects and list items** reached via the helper methods |
| `NotNull` | Object cannot be null (lets some formats save a byte; also avoids boxing value types when used without `Deduplicate`) |
| `Compact` | Format on one line (SyncJson; ignored by SyncBinary) |
| `ReadNullAsDefault` | When reading null into a value type, return `default` instead of throwing |
| `NoTypeTag` | Suppress the type tag a tagged synchronizer would otherwise write (see [Dynamic typing](#dynamic-typing-uncommitted)) |

Under the hood these calls use `BeginSubObject(name, childKey, mode, listLength)` / `EndSubObject()`, the low-level protocol every format implements. `BeginSubObject` returns `(bool Begun, int Length, object? Object)`: if `Begun` is false, the child was null, was already written/read (deduplication hit — `Object` is the existing instance), or is being skipped (Query/Schema). End-users rarely call it directly, but it is the extension point for custom object representations.

### Deduplication and cycles

With `ObjectMode.Deduplicate` (the default for object fields and list items), each distinct object is written once; later references become back-references, so shared and even *cyclic* object graphs round-trip correctly. All bundled formats support this (`SupportsDeduplication`). One requirement: when reading a type that can participate in a cycle, set `sm.CurrentObject = obj` **before** synchronizing fields that could refer back to it — this registers the instance so back-references to it can be resolved while its own fields are still being read. Strings can be deduplicated too: `sm.Sync(name, str, ObjectMode.Deduplicate)`.

### Lists, collections and dictionaries

`SyncManagerExt` provides `SyncList` for `T[]`, `List<T>`, `IList<T>`, `IReadOnlyList<T>`, `IListSource<T>`, `ICollection<T>`, `IReadOnlyCollection<T>`, `HashSet<T>`, `Memory<T>` and `ReadOnlyMemory<T>`:

- **Primitive element types** have direct overloads: `sm.SyncList("Xs", intArray)`. Byte lists are special-cased in every format (e.g. Base64 in JSON, `bytes` in protobuf).
- **Object element types** take an item synchronizer: `sm.SyncList("Siblings", people, SyncPerson, itemMode, listMode, tupleLength)`. `SyncColl` covers arbitrary `ICollection<T>` (with an `alloc` callback for custom collection types), `SyncDict` covers `Dictionary<K,V>`/`IDictionary<K,V>` (items are synchronized as `KeyValuePair<K,V>`), and `SyncMemory` covers `Memory<T>` variants. Item synchronizers can also be `ISyncField<SM,T>` or `ISyncObject<SM,T>` structs.
- Pass `listMode: ObjectMode.Tuple` with a `tupleLength` for fixed-length data; note `Memory<T>` cannot combine with `Deduplicate`.

### Introspection (mostly for readers)

These members let advanced code adapt to the format and the data:

- `SupportsReordering` — fields can be read in a different order than written (JSON, protobuf: yes; binary: no). When false, you must read every field, in order.
- `SupportsNextField` / `NextField` — the reader can report the name/ID of the next field in the stream. This enables reading fields *in stream order* (faster than reordering, no buffering) and reading string-keyed dictionaries whose keys are field names. Caveat: `NextField` reports the name as it appears in the stream, which may differ from your names if a name converter (e.g. camelCase) was used when writing.
- `GetFieldType(name, expectedType)` / `NextFieldType()` — probe whether a field exists and its approximate type, as a `SyncType` enum value (`Integer`, `Float`, `String`, `Object`, `List` and combinations like `ByteList`, plus `Null`/`Missing`/`Unknown`). Useful for reading polymorphic or irregular data.
- `IsInsideList`, `ReachedEndOfList`, `MinimumListLength`, `Depth` — list-scanning state, used mainly by the list helpers.

## Choosing a format

| | SyncJson | SyncBinary | SyncProtobuf |
|---|---|---|---|
| Output | UTF-8 JSON text | Compact custom binary | Protocol Buffers wire format |
| Self-describing | Field names stored | **Nothing stored** — schema lives in your code | Field numbers + wire types stored |
| `SupportsReordering` | yes | no | yes |
| `SupportsNextField` | yes | no | yes |
| Field identity | names (`NeedsIntegerIds` false) | order only | numbers (`NeedsIntegerIds` true) |
| Deduplication/cycles | yes (`$id`/`$ref` or `\f`/`\r`) | yes (`#`/`@` + object ID) | yes (`Ref` wrapper messages) |
| Schema output | JSON Schema draft 2020-12 | — | proto3 `.proto` file |
| Interop | Newtonsoft-compatible mode | — | verified against protobuf-net and protoc |

All three share the same helper-method shape: `Write<T>(value, syncFunc, options?)` / `Read<T>(input, syncFunc, options?)` returning `ReadOnlyMemory<byte>` / `T?`, with `I`-suffixed interface-typed variants, plus `NewWriter(...)`/`NewReader(...)` factories for streaming multiple roots (note: the `Options.RootMode` setting applies only to the one-shot helpers, not to `NewWriter`/`NewReader`).

## JSON: SyncJson

`SyncJson.Write`/`WriteI` produce UTF-8 bytes; `WriteString`/`WriteStringI` produce a `string`; `Read`/`ReadI` accept either. `NewWriter(IBufferWriter<byte>?, Options?)` and `NewReader(...)` support streaming. `SyncJson.ToCamelCase` is a ready-made name converter.

**Options** (top level): `NewtonsoftCompatibility` (default **true**), `NameConverter` (e.g. `SyncJson.ToCamelCase`), `RootMode`, `ByteArrayMode`. Writer options (`Options.Write`): `Minify`, `Newline`, `Indent`, `SpaceAfterColon`, `EscapeUnicode`, `MaxIndentDepth`, `CharListAsString`, `InitialBufferSize`. Reader options (`Options.Read`): `AllowComments` (default true), `Strict` (default false — tolerates trailing commas, lax numbers, sloppy escapes), `MaxDepth` (64), `VerifyEof`, `AllowMissingFields`, `ReadNullPrimitivesAsDefault`, and pluggable conversion callbacks (`ObjectToPrimitive`, `PrimitiveToObject`, `StringToObject`, `ObjectToNumber`, `TrueAsString`/`FalseAsString`) for coping with irregular JSON.

**Representation.** With `NewtonsoftCompatibility` on, deduplicated objects use Newtonsoft's `"$id"`/`"$ref"` properties (and `"$values"` for deduplicated arrays), type tags use `"$type"`, and byte arrays are Base64. With it off, SyncLib uses one-character metadata keys — `"\f"` (id), `"\r"` (back-reference), `"\t"` (type tag) — and byte arrays default to BAIS (`ByteArrayInString`, an ASCII-preserving encoding; `JsonByteArrayMode` selects `Array`, `Base64`, `Bais` or `PrefixedBais`). Metadata properties must be first in an object, as in Newtonsoft. Char lists can be stored as strings (`CharListAsString`).

**Performance notes.** The reader makes a single forward pass; reading fields in the written order is fastest. Out-of-order reads work but buffer the skipped fields in memory. Files over 2 GB are readable if no single out-of-order scan spans 2 GB.

**Schema mode.** `SyncJson.WriteSchema<T>(syncFunc, options?)` / `WriteSchemaString` / `NewSchemaWriter` run your synchronizer once with **no data** (`SyncMode.Schema`) and emit a JSON Schema (draft 2020-12) describing exactly what the writer would produce with the same options — including name conversion, dedup markers and byte-array encoding. Each type becomes a `$defs` entry (named after the .NET type, or the type tag if one is set), referenced by `$ref`, so recursive types work. Synchronizing one type in two conflicting ways throws. Being data-blind, it records only the code path taken with default values — conditional fields and non-default polymorphic branches are not captured.

## Binary: SyncBinary

`SyncBinary.Write`/`WriteI`/`Read`/`ReadI` plus `NewWriter(IBufferWriter<byte>, Options?)`/`NewReader(...)`; the reader can stream files of unlimited size.

The format stores **no metadata** — no field names, lengths only where needed, schema entirely in your code. That makes it very fast and compact, and correspondingly unforgiving: fields must be read in the order written, with the same (or a compatible) type, or you get an exception — or garbage.

**Encoding summary:**

- **Integers:** variable-length, 1–7 bytes for up to 49 bits (the count of leading 1-bits in the first byte selects the size), a length-prefixed "large" form (first byte `0xFE`) for anything bigger (including `BigInteger`), and `0xFF` = null. Big-endian, two's complement. Bools are integers (0/1); chars are `ushort`s.
- **Fixed-size values:** `float`/`double` are little-endian IEEE-754 (nullable variants use dedicated sentinel NaN bit patterns for null); `decimal` is the 16-byte .NET layout (null = 16×`0xFF`); bitfields are little-endian and pack adjacent fractional bytes.
- **Strings:** length-prefixed WTF-8 (UTF-8 that tolerates unpaired surrogates); interchangeable with byte arrays. **Lists:** length prefix + items; null is the single byte `0xFF`.
- **Markers** (`Options.Markers`): optional single-byte delimiters around objects (`{`…`}` / `(`…`)` alternating by depth), lists (`[`/`]`), tuples, and type tags (`'T'`). The default (`Markers.Default`) writes object start/end, list start, and type-tag markers. Markers add a little safety and readability but are part of the format: **reader and writer must use the same `Markers` setting** — it cannot be auto-detected.
- **Deduplication:** a `'#'` (first occurrence) or `'@'` (back-reference) byte plus an object ID precedes the object. Adding/removing `Deduplicate` on a field is a compatible change only while a start marker is enabled.

**Options:** `Markers`, `RootMode`, `MaxNumberSize` (default ~1 MB, caps large-format numbers), `Write.InitialBufferSize`, `Read.SilentlyTruncateLargeNumbers`, `Read.ReadNullPrimitivesAsDefault`, `Read.VerifyEof` (default true).

**Compatible type changes** (safe as data evolves): enlarging integer types (`short`→`int`→`long`→`BigInteger`); `T`↔`T?` for integers, floats, `double`, `decimal`; bool↔integer; `char`↔`ushort`; `string`↔`byte[]`; signed→unsigned *only* if no negative values were ever stored (never the reverse). Floats cannot be enlarged to doubles, and bitfields cannot change size. Everything else needs explicit versioning code.

## Protocol Buffers: SyncProtobuf

`SyncProtobuf.Write`/`WriteI`/`Read`/`ReadI`/`NewWriter`/`NewReader`, same shape as the others. The output is **genuine proto3-compatible wire format**, verified by round-tripping against protobuf-net and by compiling the generated schemas with `protoc`.

**Mapping SyncLib concepts to protobuf:**

- Field numbers come from `FieldId.Id`, or are auto-assigned 1, 2, 3… in call order. Valid range 1–536870909, excluding 19000–19999.
- The root object is a bare message body; a null root is zero bytes (a reserved `_present` field keeps a non-null-but-empty root distinguishable).
- Scalars use standard varint/I32/I64 encodings; signed integers are 64-bit two's complement varints (protobuf semantics: negative values take 10 bytes). A null nullable field is simply omitted; absent = null/default. `decimal` and `BigInteger` are length-delimited. `byte[]` is protobuf `bytes`.
- **Lists** become a nested container message whose elements are all field 1: packed for non-nullable scalars, repeated entries otherwise, with an `Opt` wrapper message per element when elements are nullable (preserving null ≠ empty-string), so null list, empty list and null elements all round-trip. Tuples are messages with fields 1…N.
- **Deduplication** wraps the value in a `Ref` message (`id` + `value`); a back-reference carries only the id. Unlike SyncBinary, the `Deduplicate` flag **must match exactly** between writer and reader.
- **Type tags** are a string in reserved field 536870911 (`_type`).
- Duplicate field numbers follow protobuf's last-wins rule; out-of-order fields are handled by indexing each message (the whole input is buffered — SyncProtobuf does not stream).

**Options:** just `MaxPayloadSize` (default 64 MB), `RootMode`, and `Write.InitialBufferSize` — protobuf semantics leave nothing else to configure (integers truncate exactly like standard protobuf parsers).

**Schema mode.** `SyncProtobuf.WriteSchema<T>` / `WriteSchemaI` / `WriteSchemaString` / `NewSchema` emit a proto3 `.proto` file that describes the wire format exactly (protoc-compilable). List/nullable/dedup wrappers appear as generated messages (`Int32List`, `StringOpt`, `PersonRef`), structurally-identical anonymous messages are merged, and conflicting uses of one type are detected and throw.

## Dynamic typing (uncommitted)

> **Status:** implemented and passing tests on the `loyc.synclib` branch, but not yet committed — details may change.

Everything above is *statically typed*: each call site names one synchronizer, so a field declared `Shape` always round-trips as the synchronizer's exact type. The dynamic-typing layer adds polymorphism — serializing `Ellipse` and `Polygon` through a `Shape`-typed field — without putting any serialization attributes on your business classes. It builds on `ISyncManager.SyncTypeTag(string?)`, which every format already implements (JSON: a `"$type"`/`"\t"` property; SyncBinary: an optional `'T'`-marked string; SyncProtobuf: reserved field `_type`).

~~~csharp
static class ShapeSync<SM> where SM : ISyncManager
{
    [TypeTag("Ellipse")]
    public static Ellipse Sync(SM sm, Ellipse? e) { ... ordinary synchronizer ... }
    [TypeTag("Polygon")]
    public static Polygon Sync(SM sm, Polygon? p) { ... }
}

// Registration, typically at startup (dynamic tier only):
SyncTypeRegistry.Default.Add(typeof(ShapeSync<>));

// In a synchronizer:
d.Border = sm.Sync("Border", d.Border, ShapeSync<SM>.Sync); // static tier: tag written & verified
d.Shapes = sm.SyncDynamicList("Shapes", d.Shapes);          // dynamic tier: dispatch by type/tag
~~~

**The three tiers:**

1. **Static tier** — the normal `sm.Sync(name, value, syncFunc)` call. If the synchronizer carries a `[TypeTag]`, the tag is written automatically (and verified when reading); `ObjectMode.NoTypeTag` suppresses it per call site. Because both tiers write the same tag, statically-written data can be read dynamically and vice versa. A tag *absent* from the stream (foreign JSON) falls back to the static type.
2. **Dynamic tier** — opt-in per call: `sm.SyncDynamic(name, value)`, `sm.SyncDynamicList(name, list)`, or compose `DynamicSync<SM,T>` (an ordinary `ISyncField`) with any collection helper. Writing dispatches on `value.GetType()`; reading dispatches on the tag from the stream. An unregistered runtime type on write throws (no silent base-class slicing). Explicit-instance overloads — `sm.SyncDynamic(name, value, synchronizers, tags)` — bypass the ambient registries for concurrent streams with different registrations.
3. **Default tier** — plain 2-argument `sm.Sync(name, value)` for arbitrary `T` resolves through `DefaultSynchronizer<SM,T>`: primitives, tuples, arrays, `List`/`HashSet`/`Dictionary`/`KeyValuePair`, enums (numeric), `DateTime`/`TimeSpan` (ISO strings), all composing recursively — and for registered user types, the dynamic machinery. Reflection runs once per (format, type) pair and is cached in a static field; there is no `Reflection.Emit`, so it is AOT-friendly.

**The two registries** (both ambient services with `Default`/`SetDefault`, copy-on-write, thread-safe, late registration supported):

- `SyncTypeRegistry` maps types → synchronizers. `Add(typeof(ShapeSync<>))` scans a class for `T Sync(SM, T)` bodies and `ISyncObject` implementations; `Add<T>(tag, delegate)` is the one-line easy mode. Discovered tags are forwarded to the ambient tag registry, so one call registers both halves.
- `TypeTagRegistry` owns the tag↔`Type` dictionary, the `[TypeTag]`-attribute convention (`virtual AttributeTagOf` — override to derive tags some other way), and the error policies: `UnknownTag` (tag not registered — throws by default; override to substitute a type or fall back) and `TagMismatch` (static read found a different tag — throws by default; override to proceed anyway).

`[TypeTag("...")]` goes on **synchronizer methods or structs, never on your data types** — business objects stay clean. Using SyncLib entirely *without* this feature is unchanged: don't register anything, don't use `SyncDynamic`, and (as before) call `sm.SyncTypeTag(tag)` manually if you want to hand-roll polymorphism.

## The implementation layer: Loyc.SyncLib.Impl

You only need this namespace to implement a new format or to squeeze out the last allocations; the extension methods above are built from these pieces. The design theme: every component is a **struct** generic over the concrete sync-manager type, so the JIT devirtualizes and inlines the whole pipeline — the delegate-based API is a thin convenience wrapper over it.

- **`ISyncField<SM,T>`** — synchronize one field/value: `T? Sync(ref SM sync, FieldId name, T? value)`. **`ISyncObject<SM,T>`** — synchronize the fields of one object (the interface form of `SyncObjectFunc`). `AsISyncField`/`AsISyncObject` wrap delegates into these interfaces.
- **`ObjectSyncher<SM,SyncObj,T>`** (factory: `ObjectSyncher.For(syncFunc, mode)`) — the bridge that turns an `ISyncObject` into an `ISyncField` by wrapping it in `BeginSubObject`/`EndSubObject`, handling null, deduplication hits, `CurrentObject` registration, and type-tag read/write/verification.
- **`SyncList<SM,T,SyncItem>`** (and a 4-parameter variant for arbitrary `ICollection<T>`) — one struct that synchronizes all the supported list types by dispatching to **`ListLoader`** (read) or **`ListSaver`/`ScannerSaver`** (write). Reading builds results through **`IListBuilder<TList,T>`** implementations (`ArrayBuilder`, `ListBuilder`, `CollectionBuilder`, `MemoryBuilder`, `SyncLibStringBuilder`); `IListBuilder.Alloc` returns the list object early so even cyclic list references resolve. Writing walks the source through an `IScanner<T>`.
- **`SyncPrimitive<SM>`** — `ISyncField` implementations for every primitive, forwarding to the manager's `Sync` overloads; **`SyncEnumAsString`**, **`SyncDateAsString`/`SyncDateAsDayNumber`**, **`SyncTimeSpanAsString`/`AsSeconds`/`AsMinutes`/`AsDays`** — the structs behind the corresponding extension methods, composable anywhere an `ISyncField` is accepted (e.g. as a `SyncList` item synchronizer).
- **`WriterStateBase`** — shared base class for binary-format writer states: buffer management over `IBufferWriter<byte>` with an inlined fast path, plus the `ObjectIDGenerator` used for write-side deduplication. (Read-side dedup is an id→object dictionary in each format's reader state; there is no public `ObjectTable` type at present.)

For format authors, the essential contract to implement is `ISyncManager` itself — the `Sync` overloads plus `BeginSubObject`/`EndSubObject` (see the extensive comments in `ISyncManager.ecs`), and the capability properties that advertise what your format can do.
