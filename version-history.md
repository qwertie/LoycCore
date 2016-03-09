---
title: "LoycCore version history"
tagline: "Gathered from commit messages. Trivial changes omitted."
layout: page
---
LoycCore and LES
----------------

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
