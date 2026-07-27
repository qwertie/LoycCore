---
title: "SyncLib Reference Manual"
tagline: "Say no to DTOs"
layout: article
toc: true
date: 18 Jul 2026
---

See also: [**Detailed API reference**](https://ecsharp.net/doc/code/namespaceLoyc_1_1SyncLib.html)

SyncLib is a .NET serialization system for real-world projects where traditional attribute-based 
serialization falls flat, forcing you to use DTOs.

Attribute-based libraries don't allow the level of control you need for

- Type conversion during serialization (Color values <=> strings, and multimaps <=> Lists)
- Nonlocal representation changes (Start + End Date <=> Start Date + Duration)
- Multiple serialization formats for backward compatibility purposes (v1/v2)
- Serializing the same type in multiple ways (storing Color as string in some places and as integer in others)

With SyncLib you can support many serializers at once in about half the lines of code as DTO-based 
transformations, while getting great performance.

The library ships in three assemblies (all in the `Loyc.SyncLib` namespace):

- **Loyc.Essentials.dll**: The core interfaces and helpers, plus the **SyncBinary** format
- **Loyc.SyncLib.SyncJson.dll**: The **SyncJson** format (JSON + JSON Schema), plus **SyncJsonDOM** for reading from a `System.Text.Json.JsonDocument` (net6.0 build only)
- **Loyc.SyncLib.SyncProtobuf.dll**: The **SyncProtobuf** format (Protocol Buffers + .proto schema)

Low-level building blocks live in `Loyc.SyncLib.Impl` (summarized [at the end](#the-implementation-layer-loycsynclibimpl)).

The synchronizer concept
------------------------

A synchronizer is a method (or lambda) with one of these signatures for some type T:

~~~csharp
T SyncObjectFunc(ISyncManager sync, T value); // simplified form (slower)
T SyncObjectFunc<SM>(SM sync, T value) where SM : ISyncManager; // preferred
~~~

Your synchronizer is called with an object to save (when writing) or a default `value` (when reading):

~~~csharp
class Person
{
    public string? Name;
    public int Age;
    public Person[]? Siblings;
}

public static Person SyncPerson<SM>(SM sm, Person? obj) where SM : ISyncManager
{
    sm.CurrentObject = obj ??= new Person(); // see "Deduplication and cycles"
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

Each `Sync` call *returns* the field's value (the saved value when writing, the loaded value 
when reading), so the same code both saves and loads. To understand synchronizers, read them
once imagining that SM is writing output (e.g. `SyncJson.Writer`), and a second time imagining 
that SM is reading a file (`SyncJson.Reader`).

For a very small performance boost, put your synchronizer(s) in a struct that implements 
`ISyncObject<SM,T>` so that SyncLib can call your synchronizers with static dispatch. It's 
also a convenient way to group synchronizers together:

~~~csharp
public struct PersonSync<SM> : 
    ISyncObject<SM, Person>,
    ISyncObject<SM, Address> where SM : ISyncManager
{
    public Person Sync(SM sm, Person? obj)
    {
        sync.CurrentObject = obj ??= new Person();
        obj.Name = sm.Sync("Name", obj.Name);
        obj.Age = sm.Sync("Age", obj.Age);
        obj.Siblings = sm.SyncList("Siblings", obj.Siblings, this);
        obj.PrincipalAddress = sm.Sync("Address", obj.PrincipalAddress, this);
        return obj;
    }

    public Address? Sync(SM sm, Address? obj)
    {
        obj ??= new();
        sm.Sync("HouseNo", ref obj.HouseNo);
        sm.Sync("Street", ref obj.Street);
        sm.Sync("City", ref obj.City);
        sm.Sync("State", ref obj.State);
        sm.Sync("PostalCode", ref obj.PostalCode);
        return obj;
    }
}
~~~

### How to write synchronizers

- When reading, construct the object (`obj ??= new Person()`) and also assign `sm.CurrentObject` 
  if the current object could ever be part of a cycle (i.e. reachable via properties of the 
  current object.)
- Pass `Sync` the same property to which you write the return value: `obj.P = sm.Sync("P", obj.P)`.
  Avoid `nameof(obj.P)` because this will change the expected property name in saved (JSON) data 
  when the property is remained, which would break backward compatibility. You can break this 
  rule if the serialized form is always transient, with name changes always synchronized between 
  sender and receiver.
- Synchronize fields unconditionally and in a fixed order to achieve compatibility with all 
  `ISyncManager` implementations. If the set of fields changes over time, use explicit versioning:
  write a version number field first, then branch on it; this is compatible with all 
  `ISyncManager`s. These rules can be relaxed if `sm.SupportsNextField` and `sm.SupportsReordering`
  return true.
- To support protocol buffers fully, include property numbers, not just names: 
  `obj.Z = sm.Sync(("Z", 3), obj.Z)` (or use `new("Z", 3)` meaning `new FieldId("Z", 3)` to avoid 
  a conversion from `(string, int)`)
- If your code must behave differently when loading vs. saving, test `sm.IsReading`/`sm.IsWriting`.
- SyncJson, SyncBinary and SyncProtobuf support object dedeuplication and cycles, but you can 
  check `sm.SupportsDeduplication` to make sure.

### How to serialize/deserialize

To read or write, call a static `Read(obj, synchronizer, options)` or 
`Write(data, synchronizer, options)` method on `SyncJson`, `SyncBinary`, or `SyncProtobuf`, unless 
your synchronizers take an `ISyncManager` argument, in which case you must call `ReadI` or `WriteI` 
instead (they cannot have the same name due to a limitation of the C# language).

Sync managers should offer these overloads (where `Options` is specific to the data format, 
e.g. `SyncJson.Options`):

    // All formats
    public static T? Read<T>(ReadOnlyMemory<byte> input, SyncObjectFunc<Reader, T> sync, Options? options = null)
    public static T? ReadI<T>(ReadOnlyMemory<byte> input, SyncObjectFunc<ISyncManager, T> sync, Options? options = null)
	public static T? Read<T, SyncObject>(ReadOnlyMemory<byte> input, SyncObject sync, Options? options = null) where SyncObject : ISyncObject<Reader, T>
    public static ReadOnlyMemory<byte> Write<T>(T value, SyncObjectFunc<Writer, T> sync, Options? options = null)
    public static ReadOnlyMemory<byte> WriteI<T>(T value, SyncObjectFunc<ISyncManager, T> sync, Options? options = null)
    public static ReadOnlyMemory<byte> Write<T, SyncObject>(T value, SyncObject sync, Options? options = null)

    // Binary formats only (SyncBinary, SyncProtobuf)
    public static T? Read<T>(byte[] input, SyncObjectFunc<Reader, T> sync, Options? options = null)
    public static T? ReadI<T>(byte[] input, SyncObjectFunc<ISyncManager, T> sync, Options? options = null)
	public static T? Read<T, SyncObject>(byte[] input, SyncObject sync, Options? options = null) where SyncObject : ISyncObject<Reader, T>

    // Text formats only (SyncJson)
	public static T? Read<T>(string json, SyncObjectFunc<Reader, T> sync, Options? options = null)
	public static T? ReadI<T>(string json, SyncObjectFunc<ISyncManager, T> sync, Options? options = null)
	public static T? Read<T, SyncObject>(string json, SyncObject sync, Options? options = null) where SyncObject : ISyncObject<Reader, T>
	public static string WriteString<T>(T value, SyncObjectFunc<Writer, T> sync, Options? options = null)
	public static string WriteStringI<T>(T value, SyncObjectFunc<ISyncManager, T> sync, Options? options = null)
	public static string WriteString<T, SyncObject>(T value, SyncObject sync, Options? options = null)

    // To create a Reader/Writer without actually reading/writing
    public static Writer NewWriter(IBufferWriter<byte>? output = null, Options? options = null)
    public static Reader NewReader(ReadOnlyMemory<byte> input, Options? options = null)
    public static Reader NewReader(IScanner<byte> input, Options? options = null)
    public static Reader NewReader(string input, Options? options = null)

To use the `IScanner<byte>` overload, one typically uses `StreamScanner` (you can read any 
`Stream` via `new StreamScanner(stream)`):

    var reader = SyncJson.NewReader(StreamScanner.OpenFile(path));
    var output = reader.Sync(null, default(T), sync, ObjectMode.Normal);

where `sync` is your `SyncObjectFunc` or `ISyncObject<,>`.

Use `WriteSchema(sync, options)` to write a JSON or protobuf schema:

	public static ReadOnlyMemory<byte> WriteSchema<T>(SyncObjectFunc<Schema, T> sync, Options? options = null)
	public static ReadOnlyMemory<byte> WriteSchemaI<T>(SyncObjectFunc<ISyncManager, T> sync, Options? options = null)
	public static ReadOnlyMemory<byte> WriteSchema<T, SyncObject>(SyncObject sync, Options? options = null) where SyncObject : ISyncObject<SchemaWriter, T>
	public static string WriteSchemaString<T>(SyncObjectFunc<SchemaWriter, T> sync, Options? options = null)
	public static string WriteSchemaStringI<T>(SyncObjectFunc<ISyncManager, T> sync, Options? options = null)
	public static string WriteSchemaString<T, SyncObject>(SyncObject sync, Options? options = null) where SyncObject : ISyncObject<SchemaWriter, T>

### Choosing a format

|                      | SyncJson | SyncBinary | SyncProtobuf |
|----------------------|----------|------------|--------------|
| Output               | UTF-8 JSON text | Compact custom binary | Protocol Buffers |
| Self-describing      | yes (by field name) | no | partly (field numbers + wire types) |
| `SupportsReordering` | yes | no | yes |
| `SupportsNextField`  | yes | no | yes |
| Field identity       | names (`NeedsIntegerIds` false) | order only | numbers (`NeedsIntegerIds` true) |
| Deduplication/cycles | yes (`$id`/`$ref` or `\f`/`\r`) | yes (`#`/`@` + object ID) | yes (`Ref` wrapper messages) |
| Schema output        | JSON Schema draft 2020-12 | None | proto3 `.proto` file |
| Interop              | Newtonsoft-compatible mode | None | verified against protobuf-net and protoc |

The ISyncManager interface
--------------------------

All formats implement `ISyncManager` and it has hundreds of extension methods. Here are the 
members meant for end users:

### Properties

#### Capabilities

    SyncMode Mode { get; }
    bool IsReading { get; }
    bool IsWriting { get; }
    bool SupportsReordering { get; }
    bool SupportsNextField { get; }
    bool SupportsDeduplication { get; }
    bool NeedsIntegerIds { get; } // true for Protobufs

If possible, use `IsReading/IsWriting` instead of `Mode`. If `Mode is SyncMode.Schema`, 
`IsReading` is true; all values "read" are default/null, and collections pretend to have one 
element.

#### Status

    int Depth { get; } // for debugging
    FieldId NextField { get; } // returns FieldId.Missing at end-of-object or if not supported
    SyncType GetFieldType(FieldId name, SyncType expectedType = SyncType.Unknown);

As implied by `NextField`, SyncLib has a concept of "which field is next in the data stream". 
Formats that support reading fields out of order (SyncJson, SyncProtobuf) typically have to 
do extra work if you skip any field while reading, because they are designed to stream in 
data from an `IScanner<byte>`. For example, if the data stream has 
`{ "A": {"subobject": true, ...}, "B": 7 }` and you read "B" without reading "A", `SyncJson` 
will save the JSON data for "A" in memory, just in case you do read it later. Simple sync 
managers like the `Person` example above normally avoid this cost. (Exception: `SyncJsonDOM.Reader` 
reads from a pre-parsed `JsonDocument`, so out-of-order reads are simple property lookups.)

`GetFieldType` gets a general type category for a field, `SyncType.Unknown` if getting types 
is not supported, or `SyncType.Missing` if the field is absent or if it is incompatible with 
`expectedType`. For example, use `expectedType = SyncType.ByteList` if you want to read a 
byte array. If reading JSON and the actual type is boolean, which cannot be interpreted as a 
byte array, `GetFieldType` returns `SyncType.Missing`. But if the JSON type is string, 
`GetFieldType` returns `SyncType.String`, as it is potentially convertible to a byte array, 
although the conversion is not guaranteed to work.

#### CurrentObject

	object CurrentObject { set; }

By default, object fields and list items are written with `ObjectMode.Deduplicate` which 
ensures each distinct object is written only once, on first encounter; later references become 
back-references, so shared and even *cyclic* object graphs round-trip correctly. All bundled 
formats support this (`SupportsDeduplication == true`).

However, when **reading** a type that can participate in a cycle, your synchronizer must set 
`sm.CurrentObject = obj` **before** synchronizing fields that could refer back to it. This 
registers the instance so back-references to it can be resolved while its own fields are 
still being read.

### Primitive Sync methods

Here are the supported primitive types, nullable or not:

    // Fun fact: the `ISyncManager` interface is defined with Enhanced C# code like this.
	define basicTypes => #splice(bool, sbyte, byte, short, ushort, int, uint, long, ulong, float, double, decimal, BigInteger, char);

    ##unroll($T in basicTypes)
    {
        $T Sync(FieldId name, $T savable);
        $T? Sync(FieldId name, $T? savable);
    }

Every `Sync` method takes a `FieldId`, a struct holding a `string Name` and `int Id`. A plain 
string converts implicitly to `FieldId`, `("Name", 5)` tuples supply an explicit ID, and a 
`Symbol` from a private `SymbolPool` can supply both. You can also write `new("Name", 5)` to 
construct `FieldId` directly (potentially faster).

SyncJson uses only the `Name`, SyncProtobuf uses only the `Id`, and SyncBinary ignores both. 
Pass `null` to read the next field in the data stream, regardless of its name.

    string? Sync(FieldId name, string? savablem, ObjectMode mode = ObjectMode.Normal);

Set `mode` to `ObjectMode.Deduplicate` to request string deduplication, if supported.

### Bitfields methods

You can request that your integers be stored in fixed-size bitfields, which can save space in 
`SyncBinary`. Sync managers that don't support this will simply ignore the `bits` and `signed`
parameters.

	define bitfieldTypes => #splice(int, long, BigInteger);

    ##unroll($T in bitfieldTypes)
    {
        /// <summary>Reads or writes a value of an integer bitfield on the current object.</summary>
        $T Sync(FieldId name, $T savable, int bits, bool signed = true);
    }

### Secret members

These members are used by extension methods; don't call them:

    BeginSubObject, EndSubObject, ReachedEndOfList, MinimumListLength, SyncTypeTag

### Extension methods for subobjects

When a property has a composite type (object/tuple), the recommended way to synchronize it is by 
providing the synchronizer delegate/object as a third argument, as shown here for `obj.Address`:

	public class Person
	{
		public string? Name { get; set; }
		public Address Address { get; set; }
    }
	public record Address(string Number, string Street, string City, string Country);


    public static Person SyncPerson<SM>(SM sm, Person? obj) where SM : ISyncManager
    {
        obj ??= new Person();
        obj.Name = sm.Sync("Name", obj.Name);
        obj.Address = sm.Sync("Address", obj.Address, SyncAddress!)!;
        return obj;
    }

You can omit the third argument by calling `SyncDyn` instead of `Sync` and using `TypeTagAttribute`
(see below), at some performance cost. Also see below about synchronizing `record`s like `Address`.

### Extension methods for collections

There are hundreds of extension methods for `ISyncManager`, most of which help read/write 
collections. It's impractical for SyncLib to support everything because its types are invariant, 
not covariant. See, a normal API like `void Write(IList<long> list)` can accept any kind of 
`IList` as a parameter, but the equivalent in SyncLib,

	public static IList<long>? SyncList<SM>(this SM sync, 
		  FieldId name, IList<long>? savable, 
		  ObjectMode listMode = ObjectMode.List, int tupleOrListLength = -1)

cannot support `List<long>`, because given `List<long> nums`, `nums = sm.SyncList("nums", nums)` 
would not compile (and the method would not know what kind of list it should create when reading).
Thus SyncLib offers separate methods for `IList<long>` `IReadOnlyList<long>`, `ICollection<long>`,
`List<long>`, and so on. The methods are specialized not just for each collection type, but also 
for each primitive type.

The extension methods have a few different names, just so that the total number of extension 
methods with the _same_ name isn't as extreme:

- SyncList: synchronize an IList, IReadOnlyList, `List<T>`, or other list type (75 overloads)
- SyncArray: synchronize an array (15 overloads)
- SyncColl: synchronize an IEnumerable, ICollection, HashSet, or other collection type (69 overloads)
- SyncDict: synchronize a IDictionary, Dictionary, or other dictionary type (6 overloads - 
  this group is simpler because items are never primitive types, but rather `KeyValuePair`)
- SyncMemory: Synchronize a Memory or ReadOnlyMemory (36 overloads)

**For collections of non-primitive types**, provide another synchronizer function or object as 
the third parameter:

    obj.Addresses = sm.SyncList("Addresses", obj.Addresses, SyncAddress);

The third parameter must conform to one of thee types; see "Types of synchronizers" below

	public delegate T SyncObjectFunc<in SyncManager, T>(SyncManager sync, [AllowNull] T value);
	public interface ISyncObject<SyncManager, T> { T Sync(SyncManager sync, T? value); }
	public interface ISyncField<SyncManager, T> { T? Sync(ref SyncManager sync, FieldId name, T? value); }

Note: Lists of bytes, bools and chars are special-cased, so that `SyncJson` can store both 
byte arrays/lists and char arrays/lists as strings (depending on the `CharListAsString`, 
`ByteArrayMode` and `NewtonsoftCompatibility` options)

### More extension methods

- `Sync` extension methods exist for `DateTime`, `DateTimeOffset`, `TimeSpan`, and custom 
  implementations of `ISyncObject<SM, T>` and `ISyncField<SM, T>`
- `SyncRef` synchronizes a field with `ref` so you needn't mention it twice: 
  `sm.SyncRef("X", ref obj.X)` (15 overloads)
- `SyncAsString(name, e)`: synchronizes an enum, DateTime, DateTimeOffset, or TimeSpan as a
  string (Newtonsoft-compatible ISO-8601 for dates; see recipe below about synchronizing enums)
- `SyncAsDayNumber(name, dateTime, asInt32)` uses OLE Automation day numbers (floating point).

Other stuff
-----------

### ObjectMode

Many Sync functions accept an `ObjectMode` as the third or fourth argument: 
`sm.Sync("Border", obj.Border, SyncShape, mode)`.

The three basic modes are Normal, List and Tuple, which are mutually exclusive; the other 
values are flags that can be combined with the three basic kinds.

| Flag | Meaning |
|---|---|
| `Normal`  | An normal object with named fields. When calling collection extension methods such as SyncList, `Normal` is treated as a synonym for `List` |
| `List`    | A list (variable number of unnamed items) |
| `Tuple`   | Fixed number of unnamed items. In `SyncBinary` the length of tuples is *not* stored, so the reader must know the `tupleLength` and pass it as the next parameter after the `ObjectMode` |
| `Deduplicate` | Avoids writing an object more than once, and enables cyclic object graphs |
| `NotNull` | Object cannot be null (avoids boxing value types when used without `Deduplicate`) |
| `Compact` | Avoids writing newlines when used with `SyncJson`; ignored by `SyncBinary` and `SyncProtobuf` |
| `ReadNullAsDefault` | When reading null into a value type, return `default` instead of throwing |
| `NoTypeTag` | Suppress the type tag a tagged synchronizer would otherwise write (see [Dynamic typing](#dynamic-typing-uncommitted)) |

Collections of non-primitive types have **two** ObjectModes: the `itemMode` (mode of the 
individual elements) and the `listMode` (mode of the list object itself), which are given 
in that order, e.g.

    public static List<T>? SyncList<SM, T, SyncObj>(this SM sync,
        FieldId name, List<T>? savable, SyncObj syncItem,
        ObjectMode itemMode = ObjectMode.Deduplicate,
        ObjectMode listMode = ObjectMode.List, int tupleLength = -1)
            where SM : ISyncManager
            where SyncObj : ISyncObject<SM, T>
	public static List<T>? SyncDynList<SM, T>(this SM sync,
        FieldId name, List<T>? savable,
		ObjectMode itemMode = ObjectMode.Deduplicate,
        ObjectMode listMode = ObjectMode.List, int tupleLength = -1)
			where SM : ISyncManager

As you can see, items are deduplicated by default but lists are not.

Recipes
-------

### Synchronizing immutable objects such as `record`

	public record Address(string Number, string Street, string City, string Country)
	{
		public static Address Empty = new("", "", "", "");
	}

    public static Address? SyncAddress<SM>(SM sm, Address? obj) where SM : ISyncManager
    {
        obj ??= Address.Empty;
        var number = sm.Sync("Number", obj.Number);
        var street = sm.Sync("Street", obj.Street);
        var city = sm.Sync("City", obj.City);
        var country = sm.Sync("Country", obj.Country);
        return sm.IsWriting ? obj : new Address(number ?? "", street ?? "", city ?? "", country ?? "");
    }

If there's a dummy object you can use during reading, like `Address.Empty` here, it is 
recommended to use `obj ??= Dummy` at the top of the synchronizer. Otherwise, you will have to 
use `?.` in each `Sync` call, e.g. `sm.Sync("City", obj?.City)`.

### Synchronizing exotic collections

For collections without builtin support, you have two options. Suppose you have a property `List`
of type `DList<X>` that implements `IReadOnlyList<X>` where `X` is not a primitive type. Then 
either

1. Shuffle data through another collection type, e.g. you can use 
   `obj.List = new DList<X>(sm.SyncList("List", obj.List, SyncX))` where `SyncX` is the 
   synchronizer for type X, assuming your collection's constructor accepts a list of `X`. 
   If the special collection has primitive elements (`DList<int>`) you can just drop the 
   synchronizer argument: `obj.List = new DList<X>(sm.SyncList("List", obj.List))`

2. Provide an allocator as the fourth argument to `SyncColl`, to show it how to create your 
   collection type directly. If you do this, C# is unfortunately unable to infer the type 
   arguments `<SM, Coll, T>` and you will get a strange error to boot:

    // Error CS1660: Cannot convert lambda expression to type 'ObjectMode' because it is not a delegate
	obj.List = sm.SyncColl("List", obj.List, SyncX, capacity => new DList<X>(capacity));

Providing the type arguments manually fixes this:

	obj.List = sm.SyncColl<SM, DList<X>, X>("List", obj.List, SyncX, capacity => new DList<X>(capacity));

But `SyncColl` also has a special eighth argument called `ignored` (which it ignores) for solving 
this problem:

	obj.List = sm.SyncColl("List", obj.List, SyncX, capacity => new DList<X>(capacity), ignored: default(Address));

### Synchronizing enums and enum lists

    obj.Enum = sm.SyncEnumAsString("Enum", obj.Enum); // sync as string

    obj.Enum = (EnumType) sm.Sync("Enum", (int) obj.Enum); // sync as int

	obj.Enums = sm.SyncList("Enums", obj.Enums, new SyncEnumAsString<SM, EnumType>()); // sync as string

    // requires `using Loyc.SyncLib.Impl`
	var syncAsInt = new AsISyncField<SM, EnumType>((ref SM sm, FieldId name, EnumType value) => (EnumType)sm.Sync(name, (int)value));
    obj.Enums = sm.SyncList("Enums", obj.Enums, syncAsInt);

This "quick and very dirty" way of synchronizing lists of enums with `AsISyncField` is inefficient. 
Doing it efficiently seems to require a special function per enum because C# doesn't allow a 
generic method to cast an `Enum` to and from `int`:

    public struct SyncEnum<SM> : 
        ISyncField<SM, EnumType1>,
        ISyncField<SM, EnumType2> where SM : ISyncManager
    {
        public EnumType1 Sync(ref SM sm, FieldId name, EnumType1 value) => (EnumType1) sm.Sync(name, (int)value);
        public EnumType2 Sync(ref SM sm, FieldId name, EnumType2 value) => (EnumType2) sm.Sync(name, (int)value);
    }

    // in your synchronizer
    obj.Enums = sm.SyncList("Enums", obj.Enums, new SyncEnum<SM>());

The more flexible but less efficient approach uses a generic `Sync` method with a type parameter 
`E: Enum` that casts through `object` (`(int)(object)enumValue`). 

### Custom primitive synchronizers

Most of the time you'll want to use normal synchronizers which are designed for composite types 
and conform to this delegate or this interface:

	public delegate T SyncObjectFunc<in SyncManager, T>(SyncManager sync, [AllowNull] T value);
	public interface ISyncObject<SyncManager, T> { T Sync(SyncManager sync, T? value); }

However, if you want to synchronize a type as a _primitive_ type, then you'll need to conform to 
_this_ delegate or interface instead:

    public delegate T SyncFieldFunc_Ref<SyncManager, T>(
        ref SyncManager sync, FieldId name, [AllowNull] T value);
	public interface ISyncField<SyncManager, T> {
		T? Sync(ref SyncManager sync, FieldId name, T? value);
	}

and the latter is preferred. For example, to serialize a `System.Drawing.Color` as a JavaScript-
like string, define this:

    public struct SyncColor<SM> : ISyncField<SM, Color> where SM : ISyncManager
    {
        public Color Sync(ref SM sm, FieldId name, Color color)
        {
            var str = sm.Sync(name, ToString(color));
            if (str == null)
            throw new FormatException("Got null when a color was expected");
            return ToColor(str);
        }

        public static string ToString(Color c) => "#" + (c.ToArgb() & 0xFFFFF).ToString("X6");
        public static Color ToColor(string? s)
        {
            if (s == null || !s.StartsWith("#"))
            throw new FormatException("Expected a color (starting with '#')");
            return Color.FromArgb(Convert.ToInt32(s.Substring(1), 16));
        }
    }

and then synchronize a field of type `Color` like this:

    obj.Color = sm.Sync("Color", obj.Color, new SyncColor<SM>());


Sync Manager implementations
----------------------------

### SyncJson

`SyncJson.Write`/`WriteI` produce UTF-8 bytes; `WriteString`/`WriteStringI` produce a `string`; `Read`/`ReadI` accept either. `NewWriter(IBufferWriter<byte>?, Options?)` and `NewReader(...)` support streaming. `SyncJson.ToCamelCase` is a ready-made name converter.

**Options** (top level): `NewtonsoftCompatibility` (default **true**), `NameConverter` (e.g. `SyncJson.ToCamelCase`), `RootMode`, `ByteArrayMode`. Writer options (`Options.Write`): `Minify`, `Newline`, `Indent`, `SpaceAfterColon`, `EscapeUnicode`, `MaxIndentDepth`, `CharListAsString`, `InitialBufferSize`. Reader options (`Options.Read`): `AllowComments` (default true), `Strict` (default false — tolerates trailing commas, lax numbers, sloppy escapes), `MaxDepth` (64), `VerifyEof`, `AllowMissingFields`, `ReadNullPrimitivesAsDefault`, and pluggable conversion callbacks (`ObjectToPrimitive`, `PrimitiveToObject`, `StringToObject`, `ObjectToNumber`, `TrueAsString`/`FalseAsString`) for coping with irregular JSON.

**Representation.** With `NewtonsoftCompatibility` on, deduplicated objects use Newtonsoft's `"$id"`/`"$ref"` properties (and `"$values"` for deduplicated arrays), type tags use `"$type"`, and byte arrays are Base64. With it off, SyncLib uses one-character metadata keys — `"\f"` (id), `"\r"` (back-reference), `"\t"` (type tag) — and byte arrays default to BAIS (`ByteArrayInString`, an ASCII-preserving encoding; `JsonByteArrayMode` selects `Array`, `Base64`, `Bais` or `PrefixedBais`). Metadata properties must be first in an object, as in Newtonsoft. Char lists can be stored as strings (`CharListAsString`).

**Performance notes.** The reader makes a single forward pass; reading fields in the written order is fastest. Out-of-order reads work but buffer the skipped fields in memory. Files over 2 GB are readable if no single out-of-order scan spans 2 GB.

**SyncJsonDOM.** If (part of) a document was already parsed by System.Text.Json — say, a web framework handed you a `JsonElement`, or you inspected the JSON to decide how to deserialize it — the `SyncJsonDOM` class reads it without re-parsing: `SyncJsonDOM.Read<T>(docOrElement, sync, options?)` / `ReadI` / `NewReader` mirror the `SyncJson` methods, and `SyncJsonDOM.Reader` understands the same conventions (`NameConverter`, `$id`/`$ref` deduplication, byte-array encodings, type tags), so it can read anything `SyncJson.Writer` wrote. Since the DOM is random-access, out-of-order reads are cheap, and a `$ref` can even refer to an `$id` that appears later in the document. Note: syntax rules are System.Text.Json's (options like `Read.Strict` and `Read.AllowComments` only influence the overloads that parse text), and it is available only in the net6.0 build, because `JsonDocument` doesn't exist in .NET Standard 2.x or .NET Framework. Prefer plain `SyncJson` when your input is bytes or text — parsing a DOM first is slower.

**Schema mode.** `SyncJson.WriteSchema<T>(syncFunc, options?)` / `WriteSchemaString` / `NewSchemaWriter` run your synchronizer once with **no data** (`SyncMode.Schema`) and emit a JSON Schema (draft 2020-12) describing exactly what the writer would produce with the same options — including name conversion, dedup markers and byte-array encoding. Each type becomes a `$defs` entry (named after the .NET type, or the type tag if one is set), referenced by `$ref`, so recursive types work. Synchronizing one type in two conflicting ways throws. Being data-blind, it records only the code path taken with default values — conditional fields and non-default polymorphic branches are not captured.

### SyncBinary

`SyncBinary.Write`/`WriteI`/`Read`/`ReadI` plus `NewWriter(IBufferWriter<byte>, Options?)`/`NewReader(...)`; the reader can stream files of unlimited size.

The format stores **no metadata** — no field names, lengths only where needed, schema entirely in your code. That makes it very fast and compact, but unforgiving: fields must be read in the order written, with the same (or a compatible) type, or you get an exception — or garbage.

**Strings** are stored in [WTF-8](https://simonsapin.github.io/wtf-8/) — UTF-8 extended so that unpaired UTF-16 surrogates encode as themselves — so *any* .NET string round-trips losslessly, even a malformed one (as of v30.3; earlier versions replaced unpaired surrogates with U+FFFD, like `Encoding.UTF8`). Well-formed strings are byte-for-byte ordinary UTF-8. The codec is the public class `Loyc.WTF8Encoding`, usable on its own.

**Compatible type changes** certain changes can be made safely: enlarging integer types (`short`→`int`→`long`→`BigInteger`); `T`↔`T?` for integers, floats, `double`, `decimal`; bool↔integer; `char`↔`ushort`; `string`↔`byte[]`; signed→unsigned *only* if no negative values were ever stored (never the reverse). Floats cannot be enlarged to doubles, and bitfields cannot change size. Everything else needs explicit versioning code.

The [detailed documentation](https://github.com/qwertie/ecsharp/blob/master/Core/Loyc.Essentials/SyncLib/Binary/SyncBinary.cs) is in doc comments.

### SyncProtobuf (Protocol Buffers)

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

Dynamic typing (experimental)
-----------------------------

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
d.Shapes = sm.SyncDynList("Shapes", d.Shapes);          // dynamic tier: dispatch by type/tag
~~~

**The three tiers:**

1. **Static tier** — the normal `sm.Sync(name, value, syncFunc)` call. If the synchronizer carries a `[TypeTag]`, the tag is written automatically (and verified when reading); `ObjectMode.NoTypeTag` suppresses it per call site. Because both tiers write the same tag, statically-written data can be read dynamically and vice versa. A tag *absent* from the stream (foreign JSON) falls back to the static type.
2. **Dynamic tier** — opt-in per call: `sm.SyncDyn(name, value)`, `sm.SyncDynList(name, list)`, or compose `DynamicSync<SM,T>` (an ordinary `ISyncField`) with any collection helper. Writing dispatches on `value.GetType()`; reading dispatches on the tag from the stream. An unregistered runtime type on write throws (no silent base-class slicing). Explicit-instance overloads — `sm.SyncDyn(name, value, synchronizers, tags)` — bypass the ambient registries for concurrent streams with different registrations.
3. **Default tier** — plain 2-argument `sm.Sync(name, value)` for arbitrary `T` resolves through `DefaultSynchronizer<SM,T>`: primitives, tuples, arrays, `List`/`HashSet`/`Dictionary`/`KeyValuePair`, enums (numeric), `DateTime`/`TimeSpan` (ISO strings), all composing recursively — and for registered user types, the dynamic machinery. Reflection runs once per (format, type) pair and is cached in a static field; there is no `Reflection.Emit`, so it is AOT-friendly.

**The two registries** (both ambient services with `Default`/`SetDefault`, copy-on-write, thread-safe, late registration supported):

- `SyncTypeRegistry` maps types → synchronizers. `Add(typeof(ShapeSync<>))` scans a class for `T Sync(SM, T)` bodies and `ISyncObject` implementations; `Add<T>(tag, delegate)` is the one-line easy mode. Discovered tags are forwarded to the ambient tag registry, so one call registers both halves.
- `TypeTagRegistry` owns the tag↔`Type` dictionary, the `[TypeTag]`-attribute convention (`virtual AttributeTagOf` — override to derive tags some other way), and the error policies: `UnknownTag` (tag not registered — throws by default; override to substitute a type or fall back) and `TagMismatch` (static read found a different tag — throws by default; override to proceed anyway).

`[TypeTag("...")]` goes on **synchronizer methods or structs, never on your data types** — business objects stay clean. Using SyncLib entirely *without* this feature is unchanged: don't register anything, don't use `SyncDyn`, and (as before) call `sm.SyncTypeTag(tag)` manually if you want to hand-roll polymorphism.


Implementation helpers: Loyc.SyncLib.Impl
-----------------------------------------

You only need this namespace to implement a new format or to squeeze out the last allocations; the extension methods above are built from these pieces. The design theme: every component is a **struct** generic over the concrete sync-manager type, so the JIT devirtualizes and inlines the whole pipeline — the delegate-based API is a thin convenience wrapper over it.

- **`ISyncField<SM,T>`** — synchronize one field/value: `T? Sync(ref SM sync, FieldId name, T? value)`. **`ISyncObject<SM,T>`** — synchronize the fields of one object (the interface form of `SyncObjectFunc`). `AsISyncField`/`AsISyncObject` wrap delegates into these interfaces.
- **`ObjectSyncher<SM,SyncObj,T>`** (factory: `ObjectSyncher.For(syncFunc, mode)`) — the bridge that turns an `ISyncObject` into an `ISyncField` by wrapping it in `BeginSubObject`/`EndSubObject`, handling null, deduplication hits, `CurrentObject` registration, and type-tag read/write/verification.
- **`SyncList<SM,T,SyncItem>`** (and a 4-parameter variant for arbitrary `ICollection<T>`) — one struct that synchronizes all the supported list types by dispatching to **`ListLoader`** (read) or **`ListSaver`/`ScannerSaver`** (write). Reading builds results through **`IListBuilder<TList,T>`** implementations (`ArrayBuilder`, `ListBuilder`, `CollectionBuilder`, `MemoryBuilder`, `SyncLibStringBuilder`); `IListBuilder.Alloc` returns the list object early so even cyclic list references resolve. Writing walks the source through an `IScanner<T>`.
- **`SyncPrimitive<SM>`** — `ISyncField` implementations for every primitive, forwarding to the manager's `Sync` overloads; **`SyncEnumAsString`**, **`SyncDateAsString`/`SyncDateAsDayNumber`**, **`SyncTimeSpanAsString`/`AsSeconds`/`AsMinutes`/`AsDays`** — the structs behind the corresponding extension methods, composable anywhere an `ISyncField` is accepted (e.g. as a `SyncList` item synchronizer).
- **`WriterStateBase`** — shared base class for binary-format writer states: buffer management over `IBufferWriter<byte>` with an inlined fast path, plus the `ObjectIDGenerator` used for write-side deduplication. (Read-side dedup is an id→object dictionary in each format's reader state; there is no public `ObjectTable` type at present.)

For format authors, the essential contract to implement is `ISyncManager` itself — the `Sync` overloads plus `BeginSubObject`/`EndSubObject` (see the extensive comments in `ISyncManager.ecs`), and the capability properties that advertise what your format can do.
