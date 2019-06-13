---
layout: default
title: Home
---
Welcome
=======

The Loyc .NET Core project is a set of general-purpose .NET libraries. LoycCore is especially focused on collections: classes, interfaces, adapters, and extension methods. 

Contributors are welcome: more unit tests, code reviews, and new features are desired, anything relatively small (under about 3000 lines of code) that fits the theme "__things that should be built into the .NET framework, but aren't__".

I strive to meet any 4 of the following 5 criteria for all code in these libraries: well-designed, well-documented, well-tested, well-written (concise/beautiful), and high-performance.

The libraries are:

- **[Loyc.Essentials.dll](http://core.loyc.net/essentials)**: a library of interfaces, extension methods, and small bits of functionality that are useful in virtually any software project. At least half of **Loyc.Essentials** is devoted to [collections](http://ecsharp.net/doc/code/namespaceLoyc_1_1Collections.html): collection interfaces, collection adapters, collection extension methods, [Linq to Lists](http://core.loyc.net/essentials/linq-to-lists.html), and even a couple of collection implementations (most notably [`DList<T>`](/collections/dlist)). The other half includes a variety of things including [`Symbol`s](http://ecsharp.net/doc/code/classLoyc_1_1Symbol.html), [localization](http://core.loyc.net/essentials/localize.html), ["message sinks"](http://core.loyc.net/essentials/messagesink.html), [a miniature NUnit clone](http://ecsharp.net/doc/code/namespaceLoyc_1_1MiniTest.html) and more.
- **[Loyc.Collections.dll](http://core.loyc.net/collections)**: a library of sophisticated data structures including ALists, VLists, and my favorite, the hash tree types `Set<T>`, `MSet<T>`, `Map<T>` and `MMap<T>`. [Learn more](collections/index.html).
- **[Loyc.Math.dll](http://core.loyc.net/math)**: Additional functionality beyond `System.Math` in [`MathEx`](http://ecsharp.net/doc/code/classLoyc_1_1Math_1_1MathEx.html); basic geometrical interfaces and structures (points, lines, rectangles); numeric interfaces and "trait" types for doing arithmetic in generic code; fixed-point structures; 128-bit integer arithmetic.
- **[Loyc.Syntax.dll](https://github.com/qwertie/LoycCore/wiki/Loyc.Syntax)**: Contains a parser for [Loyc Expression Syntax (LES)](http://loyc.net/les), and various [interfaces and base classes](http://ecsharp.net/doc/code/namespaceLoyc_1_1Syntax.html) for Loyc Languages and for users of LLLPG.
- **[Loyc.Utilities.dll](https://github.com/qwertie/LoycCore/wiki/Loyc.Utilities)**: Additional functionality that is either (A) not important enough to be placed in **Loyc.Essentials.dll** or (B) takes **Loyc.Collections.dll** as a dependency.

They are available as a [NuGet package](https://www.nuget.org/packages/LoycCore/).

Dependency tree
---------------

Low-level libraries on top:

         Loyc.Essentials
                ^   ^
                |   |
                |   +----------------+
                |                    |
         Loyc.Collections        Loyc.Math
                ^                    ^
                |                    |
           Loyc.Syntax               |
                ^                    |
                |                    |
                +---------+----------+
                          |
                     Loyc.Utilities

Transitive dependencies are not shown, e.g. Loyc.Utilities actually references all four other DLLs.

Note: the .NET 3.5 versions of these libraries depend on the compatibility library [Theraot.Core.dll](https://github.com/theraot/Theraot).
