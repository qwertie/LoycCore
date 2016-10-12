---
title: "Version history"
tagline: "Gathered from commit messages. Trivial changes omitted."
layout: page
---
LoycCore and LES
----------------

### v1.9.3: October 12, 2016 ###

- Fixed a race condition in SymbolPool.Get (issue #43)
- Added helper classes for preserving comments (`TriviaSaver`, `AbstractTriviaInjector`, `StandardTriviaInjector`)
- Changed LESv3 syntax based on user feedback (printer not yet updated)

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
