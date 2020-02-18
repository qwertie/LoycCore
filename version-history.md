---
title: "Version history"
tagline: "Gathered from commit messages. Trivial changes omitted."
layout: page
---
LoycCore and [LES](http://loyc.net/les)
------------------

### v2.7.0.3: February 17, 2020 ###

This is a transitionary release: it is probably the last release before support for .NET 3.5 and .NET 4 is dropped. It contains a bunch of breaking changes, most notably the fact that you need to add a reference to Loyc.Interfaces.dll. There will be some additional breaking changes in v2.8.0.

#### Regularization of collection interfaces

Several changes were made to the collection interfaces to make the set more complete, consistent and useful.

Each of the three common types of collection (collection, list, dictionary) now has 3-4 interfaces:

1. "AndReadOnly" interfaces (`IListAndReadOnly<T>`, `ICollectionAndReadOnly<T>`, `IDictionaryAndReadOnly<K,V>`) are to be implemented by read-only sequences that want to implement the writable interface (`IList<T>`, `IDictionary<K,V>`) for compatibility reasons.
2. "AndSource" interfaces (`ICollectionAndSource<T>`, `IListAndListSource<T>`) implement "AndReadOnly" (e.g. `IListAndListSource<T>` implements `IListAndReadOnly<T>`) but include an enhanced Loyc interface (`IListSource<T>` is the enhanced version of `IReadOnlyList<T>` and `ICollectionSource<T>` is the enhanced version of `IReadOnlyCollection<T>`).
3. An "Impl" interface (`IListImpl<T>`, `ICollectionImpl<T>`, `IDictionaryImpl<K, V>`) is the main interface that editable Loyc collection types should implement.
4. Extended interfaces (`ICollectionEx<T>`, `IListEx<T>`, `IDictionaryEx<T>`) are variations that add extra functionality to allow sophisticated collection types to offer higher performance for some operations.

Categories 1-3 exist mainly for the purpose of disambiguation and you should generally NOT create variables of these types, as explained in the [documentation of `IListImpl<T>`](http://ecsharp.net/doc/code/interfaceLoyc_1_1Collections_1_1IListImpl.html).

#### Loyc.Interfaces.dll

Added Loyc.Interfaces.dll (available on NuGet) to hold (almost) all interfaces defined in Loyc.Essentials.dll, Loyc.Syntax.dll, etc. Aside from interfaces, enums, and delegates, it also contains a small number of essential structs and classes, most notably `Symbol`, `UString`, `Maybe<T>`, and `Localize`. These are types that the interfaces refer to, or that the concrete types themselves depend on. This new library allows you to use Loyc interfaces without taking a dependency on the heavier Loyc.Essentials library. However, this initial release is heavier than I would like at 84KB, as it includes some concrete types that I would rather not include (they are included because `Symbol`, `UString`, `Maybe<T>`, and/or `Localize` depend on them, directly or indirectly). Some of these types will be moved back to Loyc.Essentials in a future version.

- Marked `IAddRange` and `IListRangeMethods` as `<in T>`.
- Added `ICollectionSource<T>` and added it as a base of `ISetImm`
- Added `ICollectionAndReadOnly<T>`, `IListAndReadOnly<T>`, `IDictionaryAndReadOnly<T>`
- Added `ICollectionAndSource<T>`
- Added `IDictionaryEx` interface
- Added `IOptimize` as common subinterface of `IAutoSizeArray<T>` and `INegAutoSizeArray<T>`
- `IListAndListSource<T>` now includes `ICollectionSource<T>` (which has `CopyTo` and `Contains` methods.)

#### Loyc.Essentials

Changes that are most likely to be breaking:

- Rename `BaseDictionary` to `DictionaryBase` for consistency.
- Eliminated `StringSlice` (`UString` now implements its interface, `IRange<char>`)
- A new namespace [`Loyc.Collections.MutableListExtensionMethods`](http://ecsharp.net/doc/code/namespaceLoyc_1_1Collections_1_1MutableListExtensionMethods.html) has been added specifically to contain extension methods for `ICollection<T>` and `IList<T>` that are possibly ambiguous when included in the same namespace as extension methods for `IReadOnlyCollection<T>` and `IReadOnlyList<T>`. This is needed due to a design flaw in .NET and C#. [Issue #84](https://github.com/qwertie/ecsharp/issues/84) describes the problem in more detail. Most of the extension methods in this namespace are likely to be deprecated soon.

Other changes:

- Removed `ICollectionEx<T>.RemoveAll` which works well enough as an extension method (none of the existing collection types accelerate it, though AList could theoretically do so)
- Added extension methods for `IDictionary` and `IDictionaryEx`: `GetAndRemove`, `AddRange` with `DictEditMode` parameter
- Added `SetExt` extension methods for `ISet<K>`
- Added `IDictionarySink` and `IDictionaryAndReadOnly`.
- Added extension method `AddIfNotPresent` for `IList<T>`
- Added extension method `AsReadOnlyDictionary`  with `SelectDictionaryFromKeys` adapter
- Added extension methods for `IDictionary<K,V>`: `TryGetValue(K key)`, `AddRange`, `SetRange`, `RemoveRange`
- Added more `LinqToLists.Select` extension methods for `ICollection`/`IReadOnlyCollection`, with corresponding adapters `SelectCollection` and `SelectReadOnlyCollection`.
- Edit `BDictionary` and `MMap` to support `IDictionaryEx`.
- Edit `InvertibleSet` to support new `CopyTo` method.
- Added `ByteArrayInString` class, which encodes and decodes BAIS (Byte Array In String) encoding. BAIS preserves runs of ASCII characters unchanged. It is useful for debugging (since ASCII runs are visible) and for conversion of bytes to JSON.
- Added `G.DecodeBase64Digit`, `G.EncodeBase64Digit`.
- Added overloads of `TryPopFirst` and `TryPopLast` in `UString` and `RangeExt`.

#### Loyc.Collections:

- `BMultiMap` now accepts `null` values for the key and value comparator. A null key comparer means "use the default key comparer", while a null value comparer means "pretend all values are equal". If you use the `(null, null)` constructor, you should avoid calling `this[key].Remove(value)` or `this[key].Contains(value)`, since these methods will treat _any_ value as a match.

#### Loyc.Syntax:

- Replaced `#trivia_` prefix with `%` ([issue #61](https://github.com/qwertie/ecsharp/issues/61))
- LES2 and LES3 precedence tables have changed ([issue #87](https://github.com/qwertie/ecsharp/issues/87))
- LES3: Support `:` as prefix operator (not supporting it earlier was accidental)
- LES3: Add `initially` (alongside `finally`) as a continuator
- Added `LNode.FlattenBinaryOpSeq()` 
- Added `CodeSymbols.Matches` (`'=~` operator)
- Rename `#usingCast` to `'using` (`CodeSymbols.UsingCast`)
- Rename `#namedArg` to `'::=` (alternate name: `<:`) (the name `CodeSymbols.NamedArg` is unchanged)
- Rename `#of` to `'of` (`CodeSymbols.Of`) ([issue #96](https://github.com/qwertie/ecsharp/issues/96))
- Rename `#tuple` to `'tuple` (`CodeSymbols.Tuple`) ([issue #96](https://github.com/qwertie/ecsharp/issues/96))
- Rename `#is/#as/#cast` to `'is/'as/'cast`  (`CodeSymbols.Is/As/Cast`) ([issue #96](https://github.com/qwertie/ecsharp/issues/96))
- Rename `#default` to `'default`  (`CodeSymbols.Default`) even for the label form, `default:` ([issue #96](https://github.com/qwertie/ecsharp/issues/96))
- Rename `#sizeof/#typeof` to `'sizeof/'typeof` (`CodeSymbols.Sizeof/Typeof`)
- Rename suffix operators (`x[], x++, x--`) to have a consistent prefix of `'suf` ([issue #96](https://github.com/qwertie/ecsharp/issues/96))
- Rename `CodeSymbols.Neq→NotEq`, `NodeStyle.Statement→StatementBlock`
- Split `#new` into `'new` operator and `#new` attribute (`CodeSymbols.New` and `CodeSymbols.NewAttribute`)
- Split `#switch` into `'switch` operator and `#switch` statement. `CodeSymbols.Switch` was split into `CodeSymbols.SwitchStmt` and `CodeSymbols.SwitchExpr`. Note: EC# parser does not yet support the switch expression.
- Increase block size in `BaseLexer`, mainly to reduce branch mispredictions
- Deleted `IToLNode`. `ILNode.ToLNode` is now an extension method only, not a core part of the interface.

### v2.6.9: January 21, 2020 ###

Loyc.Essentials:

- Enable compiler optimizations (oops)

Loyc.Collections:

- ALists:
  - Added `CountSizeInBytes()` methods in all variants of AList (see also `ListBenchmarks.CountSizeInBytes` methods implemented for `List<T>, Dictionary<K,V>, SortedDictionary<K,V>`, and `LinkedList<T>`)
  - Added convenience method `SparseAList<T>.AddSpace(count)`
  - `AList` and `SparseAList`: Change how node splitting works for better bulk-add performance. Specifically, when splitting to add an item to the end of a node, the new left node will be 100% full and the right node will contain only one item, so that repeated calls to Add will naturally build a sequence of full nodes.
  - Optimize AList insertions by redistributing adjacent children in bulk
  - Optimize `AList.AddRange`
  - Optimize `AList<T>.this[index]` for small lists
  - Switch internal list from `InternalDList` to `InternalList` (`InternalDList` may have slightly faster insert/remove, but other operations are faster, most importantly `this[int]`.)
  - Default inner node size is now 64 to match leaf size
  - Node classes: Add debug/test properties `Children` and `Leaves`
  - Hide unimportant properties in debugger
- Optimize enumerators of InternalList, InternalDList and AList
  - InternalList: `yield return` measured to be slow. Avoided it.
  - DList/AList: Factor out "rare" parts to encourage inlining.

Loyc.Utilities:

- Add `AListSumTracker` and `AListStatisticTracker` classes with extension methods on `AListBase<K,T>`

### v2.6.8: May 12, 2019 ###

- Introduced .NET Standard 2.0 versions. NuGet package now contains four builds: .NET 3.5, .NET 4.0, .NET 4.5 and .NET Standard 2.

### v2.6.7: March 17, 2019 ###

- LES: Make `* /` and `<< >>` miscible. Precedence of `<< >>` is between `+ -` and `* /` so `x << 1 - 1` means `(x << 1) - 1`.
- LES: Keep `& | ^` immiscible with `== != > <`, but give them higher precedence than comparison operators in LES3 only.

### v2.6.6: March 16, 2019 ###

- Loyc.Syntax issue #61: `%` is now recognized as a trivia marker by `CodeSymbols.IsTriviaSymbol`, `LNode.IsTrivia`, `LNodeExt.IsTrivia`.
- LES: Add triangle operators `|> <|`
- LES bug fix: fix precedence of `..<` to match `..`
- LES3: Allow nonparenthesized attributes to have an argument list (e.g. `@attr(args) expr`)

### v2.6.5: February 17, 2019 ###

Loyc.Syntax:

- Design fix: Change `ILNode.Target` to have a more appropriate return type.
- Added convenience method for printing `ILNode` in `Les2LanguageService` and `Les3LanguageService`

### v2.6.4: September 18, 2018 ###

- Loyc.Collections: Add support for priority queues: `MinHeap`, `MaxHeap`, `MinHeapInList`, `MaxHeapInList`.

### v2.6.3: July 23, 2018 ###

Loyc.Syntax:

- Edit the default token-to-string method `TokenExt.ToString()` so it works more sensibly on non-LES tokens that are still based on `TokenKind`.
- Swap int value of `TokenKind.Spaces` with `TokenKind.Other` so that lexers oblivious to `TokenKind` will typically have integer values in the `TokenKind.Other` region.
- `Token.ToSourceRange` and `Token.Range` were duplicate methods; removed the first.

### v2.6.2: October 1, 2017 ###

- LESv3: Tweaked the precedence of `->` and `<-`. These operators can now be mixed with logical and bitwise operators (`|`, `&&`, `|`, `^` , `&`, `==`, `>`, etc.) and are right-associative.
- v2.6.2.1: Bug fix in `SparseAList.NextHigherItem`, `NextLowerItem` when `index` is negative or `int.MaxValue`.

### v2.6.1: September 28, 2017 ###

#### LES

- Made several changes to LESv3 and LESv2 as discussed in [issue #52](https://github.com/qwertie/ecsharp/issues/52):
  - v3: Added token lists like `'(+ x 1)` to allow custom syntax (comparable to s-expressions)
  - v3: `.dotKeywords` will be stored as `#hashKeywords`
  - v3: `#` is officially an ID character (but was already treated as such)
  - v3: `.keyw0rds` can now contain digits
  - v3: Hexadecimal and binary floating-point numbers no longer require a `p0` suffix
  - v2/v3: The precedence of non-predefined operators will now be based on the last punctuation character, rather than the first punctuation character. If applicable, the first and last punctuation character still determines the precedence (e.g. `>%>` will still have the same precedence as `>>`)
  - v3: The precedence of `::` has been raised to match `.`
  - v3: `x*.2` will now be parsed as `x * .2` rather than `x *. 2`, but `0..2` will still be parsed as `0 .. 2`.
  - v2/v3: The precedence of arrow operators like `->` has been lowered. The parser treats their precedence sa above `&&` but below `| ^`
  - v3: Modified the set of continuators.

### v2.6.0: August 30, 2017 ###

#### Loyc.Essentials

- Added `G.True(T)`, `G.True(Action)`, `G.Swap(ref dynamic, ref dynamic)`

#### Loyc.Syntax

- LESv2/v3: Operator `..` now has precedence above `+- * /`.
- LESv2/v3: Change `!!` from an infix to a suffix operator based on Kotlin's `!!` operator. Add `!.` operator with precedence of `.`.
- LESv3: Negative numbers are now written `n"-123"` instead of `-123`, (see [blog post](http://loyc.net/2017/negation-blues.html)). Lexer tentatively accepts −123 (en dash).
- LESv3: Enable printer to print `- -a` and `a. -b` instead of `-`'-`(a)` and `a.(@@ -b)`.
- Changed symbols `#[]`, `#[,]`, `#in` to `'[]`, `'[,]`, `'in` for consistency with other operators.

### v2.5.1: January 19, 2017 ###

#### Loyc.Syntax

- Bug fix: `ParsingService.ParseFile` could cause `IOException` that says "Cannot access a closed file"
- `BaseLexer`/`BaseParser`: Renamed the `Error` method that is called by `Match`/`MatchExcept` to `MatchError` and added an anti-infinite-loop mechanism.

### v2.5.0: January 13, 2017 ###

#### Loyc.Essentials

- Renamed `ISinkCollection` => `ICollectionSink`, `ISinkArray` => `IArraySink`, `ISinkList` => `IListSink` for consistency with `IMessageSink` and `IListSource`

#### Localization

- Added `Localize.UseResourceManager()`
- Methods of `Localize` that accept a symbol representing the string to translate were renamed from `Localized` to `Symbol`.
- Introduced "auto-fallback" behavior in `ThreadLocalVariable`. `Localize.Localizer` and `Formatter` use it to mitigate .NET's lack of thread-local propagation.

#### Message sinks (logging)

- `MessageSink`: Added extension methods `Error`, `Warn`, `Debug`, etc. Added customizable `ContextToString` delegate to replace `LocationString`. Added `WithContext`.
- Removed setter of `MessageSink.Default` (use `SetDefault` instead)
- Wrappers `SeverityMessageFilter` and `MessageSplitter` now have an optional `TContext` type parameter.
- `ConsoleMessageSink`, `TraceMessageSink` and `NullMessageSink` have singleton `Value` properties now.
- Added `MessageSinkWithContext<TContext>` wrapper for `IMessageSink`.
- Changed argument order of `TraceMessageSink` for consistency with other classes
- Removed `Severity.Detail`, replacing it with a separate `Detail` version of each member of `Severity`.
- Added `includeDetails` parameter to constructor of `SeverityMessageFilter` in the hope of drawing attention to the possibility of accidentally excluding details, while making some older code behave correctly at the same time.
- Renamed `MessageSplitter` to `MessageMulticaster`. Renamed some method parameters.

### v2.4.2: January 8, 2017 ###

- Rename `LinqToCollections` to `LinqToLists`.
- Rename some rarely used members of `Severity` to form a group of "frequency signals": `Common`, `Uncommon`, `Rare`.
- Add extension methods for `IServiceProvider`/`IServiceContainer` (`System.ComponentModel.Design`) in static class `Loyc.ServiceProvider`.

#### LESv3 (prerelease) changed:

- Eliminated block-call expressions
- Eliminated single-quoted operators
- Keywords now have a separate token type
- Syntax of keyword expressions changed. Initial expr now optional; newline allowed before braces/continuators; any expression is allowed after a continuator.
- Parser: introduced token lists (`VList<LNode>`) and eliminated token tree literals (`TokenTree`)
- Parser: edited to reduce output size
- Parser: tweaked error handling
- Parser: `@@` now acts like an attribute instead of only being allowed after `(`
- Parser now requires a space after word operators and "combo" operators

### v2.4.0.1: December 26, 2016 ###

- Changed LES generic syntax so that `a.b!c` is parsed as `a.(b!c)` instead of `(a.b)!c`.

### v2.3.3: December 21, 2016 ###

- LESv3: added syntax `(@@ expr)` to group an expression without adding `#trivia_inParens`
- LESv3 parser now requires whitespace after an attribute.
- Bug fix: LES2 and LES3 printers now print ``@`'.`(a, -b)`` correctly.
- Bug fix in `LNodeExt.MatchesPattern`: `Foo(...)` did not match pattern `$x(...)`
- Renamed `IReferenceComparable` to `IReferenceEquatable`
- Moved `Reverse()` extension method to `LinqToCollections`.

### v2.3.2: December 12, 2016 ###

- Changed return type of `MessageSink.SetDefault` as intended to be done earlier.
- Rename `MathF8`, `MathF16`, etc. to `MathFPI8`, `MathFPI16`, etc.

### v2.3.1: December 11, 2016 ###

Loyc.Essentials was split into Loyc.Essentials and Loyc.Math. Loyc.Math contains

- Most of the math stuff, including `Math128`, `MathEx`, the fixed point types (`FPI8`, `FPI16`, etc.), `Maths<T>`, and the math helper structs (`MathI`, `MathD`, etc.)
- Most of the geometry stuff, including `BoundingBox<T>`, `LineSegment`, `Point<T>`, and `Vector<T>`
- `NumRange<Num,Math>`

However, Loyc.Essentials retains most of the math interface definitions.

Loyc.Syntax and Loyc.Collections reference Loyc.Essentials but not Loyc.Math.

Some static classes were split between Loyc.Essentials & Loyc.Math:

- `Range` was split. The class `Range` went to Loyc.Math and contains methods that return `NumRange`. The `IsInRange` and `PutInRange` methods went class `G` in `Loyc.Essentials`
- `MathEx` was split. Most methods remained in `MathEx` which went to Loyc.Math. The methods `Swap` and `SortPair` went to class `G`. While `CountOnes`, `Log2Floor`, `ShiftLeft` and `ShiftRight` are still in `MathEx`, certain overloads are implemented in `G` or duplicated in `G` because they are needed by Loyc.Collections and Loyc.Syntax.

Other changes:

- The rarely-used `CPTrie` types and `KeylessHashtable` were moved from Loyc.Collections to Loyc.Utilities
- Some geometry stuff was moved from Loyc.Utilities to Loyc.Math.
- The `ParseHelpers` class was split between Loyc.Essentials and Loyc.Syntax, with the conversion-to-string stuff going to `PrintHelpers` in Loyc.Essentials and the conversion-from-string stuff going to `ParseHelpers` in Loyc.Syntax.
- Certain classes like Localize were changed to follow the Ambient Service Pattern more closely.

### v2.2.0: December 7, 2016 ###

- Tweaked Ambient Service Pattern
    
    - Renamed `MessageSink.Current` to `MessageSink.Default`
    - Introduced `IHasMutableValue<T>`
    - Renamed `PushedTLV<T>` to `SavedValue<T>`, now operates on `IHasMutableValue<T>`
    - Removed `G.PushTLV()`

### v2.1.0: December 3, 2016 ###

- Eliminated `LNodePrinter` delegate, replacing it with `ILNodePrinter`. `LNodePrinter` now holds standard extension methods of `ILNodePrinter`.
- `IParsingService` no longer includes a printing capability; language services implement `ILNodePrinter` separately.
- Renamed `LesNodePrinter` to `Les2Printer`, `LesParser` to `Les2Parser`, and so on for `LesLexer` and LESv2 unit tests.
- `LinqToCollections`: Added `TakeWhile()` and `SkipWhile()` extension methods for  IList`, `IListSource`, `INegListSource`.
- Added extension method `INegListSource<T>.TryGet(index)`
- Added one-param constructor to `MacroProcessor` and switched order of params to other constructor for harmony.

### v2.0.0: November 23, 2016 ###

- `IParsingService.Print` has changed as follows:
    - Old: `string Print(LNode node, IMessageSink msgs, object mode, string indentString, string lineSeparator);`
    - New: `string Print(LNode node, IMessageSink msgs, ParsingMode mode, ILNodePrinterOptions options);`
- Introduced `ILNodePrinterOptions` interface and `LNodePrinterOptions` class. This interface is shared among all languages; each language also has its own options type with extra options. The options object eliminates the need for users to create printer objects, so the constructors of the LES/EC# printers have gone internal.
- We now avoid using `SourcePos` for the context of `IMessageSink.Write()`; `SourceRange` is now used instead.
- `IParsingService.Parse` now preserves comments by default, if possible.
- `LNode.Equals` now has a `CompareMode` parameter which can be set to ignore trivia.
- Bug fix: `InParens()` no longer creates trivia with a range covering the entire node (it confuses the trivia injector)
- Bug fix: `IndexPositionMapper` unit tests were not set up to run, and it was broken
- Eliminated `NodeStyle.Alternate2`; added `NodeStyle.InjectedTrivia` which is applied to trivia attributes by `AbstractTriviaInjector`.
- `LesNodePrinter` now includes context when an unprintable literal is encountered.
- Introduce generic base interfaces of `IMessageSink`.

### v1.9.6: November 23, 2016 ###

- Eliminated rarely-used optional arguments of some `LNode` creation methods.
- `LinqToCollections`: Added `Select(this IList<T>, ...)` and `FirstOrDefault(this IList<T>, T)`.
- Bug fix in `LNodeFactory` that messed up trivia injector for EC#

### v1.9.5: November 14, 2016 ###

- LESv2 and LESv3 can now preserve comments and newlines and round-trip them to their printers.
- Added a few extension methods, most notably `StringBuilderExt`.
- Introduced `ILNode` interface for objects that want to pretend to be Loyc trees. Added `LNode.Equals(ILNode, ILNode)`.
- LES printers can now print `ILNode` objects directly, but most other functionality still requires `LNode` and this is not planned to change.
- Changed the way trivia works (new `#trivia_trailing` attribute) and improved the way the LES2 printer prints trivia
- Tweaked the way the trivia injector talks to its derived class.
- `IParsingService`: Added overload `Print(IEnumerable<LNode>, ...)`.
- `LNodeFactory`: Added `TriviaNewline`, `OnNewLine()`
- `LNodeExt`: Added `GetTrivia()`
- `EscapeC` & `EscapeCStyle`: Changed `EscapeC.Default`; added `UnicodeNonCharacters` and `UnicodePrivateUse` options. Single quotes are no longer escaped by default but non-characters and private characters are.
- Eliminated `SelectNegLists` and `SelectNegListSources`, which most likely no one was using. Renamed `NegView` to `AsNegList` for consistency with `AsList` etc. Added methods to `LinqToCollections` for `INegListSource`.
- Renamed `Front` to `First` and `Back` to `Last` in the `Range` interfaces.
- Changed API of `Les3PrettyPrinter`
- Updated LESv3 printer to support most of the recent changes to LESv3. Semicolons are no longer printed by default.
- LESv3 now allows `\\` to end single-line comments, to make it easier for the printer to avoid newlines in illegal places.

### v1.9.4: October 25, 2016 ###

- `IParsingService`: added `CanPreserveComments` property and comment preservation flag when parsing
- Improved trivia injector (`AbstractTriviaInjector`); added `NodeStyle.OneLiner`
- Added `UString.EndsWith()`

### v1.9.3: October 12, 2016 ###

- Fixed a race condition in `SymbolPool.Get` (issue #43)
- Added helper classes for preserving comments (`TriviaSaver`, `AbstractTriviaInjector`, `StandardTriviaInjector`)
- Changed syntax of LESv3, partly in response to the [survey results](https://goo.gl/forms/XYRV1NrxfOB4IHNu1): Newline is now a terminator; added `.keywords`; removed juxtaposition operator; added new kinds of infix operator based on identifiers (e.g. `x s> y`, `a if c else b`); colon is allowed as a final suffix on any expression. (printer not yet updated)

### v1.9.2: September 3, 2016 ###

- Introduced LESv3 parser, printer, unit tests and parsing service (`Les3LanguageService`), plus a pretty-printer (`Les3PrettyPrinter`) with colored HTML and console output.
- Add `BigInteger` literals to LESv2 and LESv3 (Issue #42) via `z` suffix
- Tweaked LESv2 and LESv3 precedence rules. Most code will be unaffected; as LES has few users, it seems better not to fork `LesPrecedence` and `LesPrecedenceMap` for v3.
- `LNode`: changed definition of `HasSpecialName` due to LESv3 idea to use '.' to mark keyword-expressions.
- `LNode:` added `PlusAttrBefore()`, `PlusAttrsBefore()`
- `NodeStyle.DataType` is unused, so changed name to `Reserved` (for future use)
- `ParseHelpers`: `EscapeCStyle`/`UnescapeCStyle`: added support for 5-digit and 6-digit unicode code points and changed APIs to be `UString`-centric.
- Added `StringExt.AppendCodePoint`, `UString.Left`, `UString.Right`. Allow `UString -> Symbol` conversion. Fixed bugs in `StringExt.EliminateNamedArgs`. Eliminated bitwise-not-on-error behavior when `UString` decodes a code point, because the behavior wasn't wanted in LES parser.
- Add `ParseHelpers.IntegerToString` and `ParseNumberFlag.SkipSingleQuotes`
- Tweaked `NodeStyle` to have named values for common literal styles (hex, triple-quoted, etc.)
- Bug fix in `MiniTest`: failure was ignored by overloads that took a string message

### v1.9.0: July 26, 2016 ###

- Names of operators in LES and EC# now start with an apostrophe (`'`)
- Also, `suf` in operator names changed from being a prefix to a suffix
- Renamed `CodeSymbols.*Set` to `CodeSymbols.*Assign`
- Removed 'Python mode' from LES; tweaked the lexer

### v1.8.1: June 13, 2016 ###

- Bug fix in LES printer: `/*suffix comments*/` were printed incorrectly (with no content).

Minor changes:

- Loyc.Essentials: Changed illogical return value of `LCInterfaces.IndexOf()` to be nullable.
- Deleted `G.Assert` in favor of `Contract.Assert()` compatibility method in .NET 3.5 builds
- Bug fix: `IndentTokenGenerator` (and LES parser) crashed in case of empty input
- Bug fix in `LNode`: implemented `IReadOnlyCollection.Count`

### v1.7.6: 2016 ###

- Add extension methods for `LNode`: `Without`, `WithoutAttr`

### v1.7.5: April 29, 2016 ###

- VLists: added `SmartSelectMany`; replaced dedicated `WhereSelect` with one based on `Transform`; deleted `Select<Out>`; renamed `Where` to `SmartWhere`. Replaced a few `Predicate<T>` with `Func<T, bool>` for consistency with LINQ-to-Objects

### v1.7.4: April 18, 2016 ###

- `IParsingService`: changed `inputType` parameters to use `ParsingMode`, a new extensible enumeration. Added additional modes.
- Helper methods for parsing and printing tokens and hex digits have been moved from `Loyc.G` to `Loyc.Syntax.ParseHelpers`
- Renamed `USlice(this string...)` to `Slice`
- Removed `G.Pair` which was a duplicate of `Pair.Create`

### v1.7.3: April 18, 2016 ###

- Bug fix in MiniTest: some assert methods ignored user-defined message
- Bug fix in `LastIndexWhere()` extension method

### v1.7.2: April 1, 2016 ###

- Extension methods: added `Slice(this NumRange)`, and non-allocating overloads of `IsOneOf()`
- `LNode`: Added new `ReplaceRecursive` overload that can delete items.
- `LNode` perf: All call nodes now cache their hashcode.

### v1.6.0: Mar 9, 2016 ###

- Renamed `Range.Single` => `ListExt.Single`, `MathEx.InRange` => `Range.PutInRange`, `In.*` => `Range.*`, `Range.Excl` => `Range.ExcludeHi`, `Range.Incl` => `Range.Inclusive`

### v1.5.1: Mar 5, 2016 ###

- Removed `IntRange`; replaced by `NumRange<N,M>` to support the EC# `lo..hi` and `lo...hi` operators; added `Floor` and `Ceiling` to Maths types as needed by `NumRange`.

### v1.5.0: Mar 2, 2016 ###

- Renamed `RVList` to `VList`, and renamed `RWList` to `WList`.
- `LNodeFactory`: Renamed `F._Missing` to `F.Missing` (and the less-oft-used `LNodeFactory.Missing` to Missing_ to avoid a name collision).
- `LNode`: Added overloads: of `LNode.Calls(string)`, `LNode.IsIdNamed(string)`, `LNode.List(IEnumerable<LNode>)`. 
- `LNodeExt`: Added `WithoutNodeNamed` extension method.
- Renamed `LineAndPos` to `LineAndCol`

### v1.4.1: Feb 28, 2016 ###

- (jonathan.vdc) LES: Added named float literals for infinity and NaN.

### v1.4.0: Aug 25, 2015 ###

- Renamed `MessageHolder.Message` to `LogMessage` (top-level class); introduced a wrapper exception `LogException`.

### July updates: forgot to raise version number ###

- LESv2: working and mostly tested! All LES-based parsers, lexers and unit tests throughout the codebase have been updated to use the new syntax. JSON compatibility was also added.
- Combined `TokenKind.Number`, `TokenKind.String`, `TokenKind.OtherLit` into a single thing, `TokenKind.Literal`, and removed all distinctions between literals from the parsers for LES, EC# and LLLPG.
- Split the symbol for indexing and array types into two separate symbols `_[]` (IndexBracks) and `[]` (Array), where previously `[]` (Bracks) was used for both.
- Bug fix: updated `LNodeFactory`, `StdSimpleCallNode`, `StdSimpleCallNodeWithAttrs` to support custom source range for `Target`.
- LES: Parser rewritten into a completely whitespace-agnostic version with a "juxtaposition operator" instead of superexpressions. However, I don't think I'll keep this version, it's too limiting.
- LES: Removed `TokensToTree` from lexer pipeline.
- LES: Language made simpler by removing all operators containing backslash.

### v1.3.2: Jun 21, 2015 ###

- Refactored LES parser to use the new `BaseParserForList` as its base class. Added property `BaseParser.ErrorSink` to match `BaseLexer` (to eliminate the need for `ParserSourceWorkaround`)
- Moved all unit tests from LoycCore assemblies to LoycCore.Tests.csproj (renamed from Tests.csproj).
  Savings: about 20K in Loyc.Essentials, 61K in Loyc.Collections, 55K in Loyc.Syntax, 25K in Loyc.Utilities.
- LES [(v1)](https://github.com/qwertie/LoycCore/wiki/LESv1): "Python mode" (ISM) installed and tested
- Added `IIndexToLine` as part of `ILexer<Token>` because certain lexers don't have a `SourceFile`, and `BaseLexer` implements `IndexToLine` anyway
- Implemented and tested `IndentTokenGenerator`
- Changed `IDeque` interface to use `Maybe<T>` and eliminated most of the extension methods
- `list.TryGet(i)` extension methods for lists now return `Maybe<T>.NoValue` instead of `default(T)` on fail
- Defined `IHasValue<T>`
- Moved the benchmarks to their own project, LoycCore.Benchmarks.
- Bug fix: `NullReferenceException` in `AListIndexer.VerifyCorrectness`

### v1.3.1: Jun 14, 2015 ###

- `ILocationString.LocationString` changed to `IHasLocation.Location`; now returns `object` for greater generality
- Some names changed in `Maybe<T>`. Added `Or()`, `AsMaybe()` & `AsNullable()`.
- `ILexer<Token>` now has a type parameter; `NextToken()` now returns `Maybe<Token>` instead of `Token?` so that `Token` is not required to be a struct.
- Added `BaseILexer<CharSrc,Token>`, and installed it as the new base class of `LesLexer`
- Added `BaseLexer.ErrorSink` property; default behavior is still to throw `FormatException`
- `LesLexer` no longer produces `Spaces` tokens.
- `LesNodePrinter` bug fix: /*comments*/ were printed twice, once as comments and once as attributes
- Bug fix in `ArraySlice.Slice(), ListSlice.Slice()`

### v1.3.0: May 27, 2015 ###

I'm not attempting to make release notes this far back, although the vast majority of code and features were written well before v1.0.
