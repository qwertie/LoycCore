---
title: "Version history"
tagline: "Gathered from commit messages. Trivial changes omitted."
layout: page
---
LoycCore and LES
----------------

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
