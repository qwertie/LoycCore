---
layout: default
title: Home
---
Welcome
=======

The Loyc Core project is a set of general-purpose .NET libraries. LoycCore is especially focused on collections: classes, interfaces, adapters, and extension methods. 

Contributors are welcome: more unit tests, code reviews, and new features are desired, anything relatively small (under about 3000 lines of code) that fits the theme "__things that should be built into the .NET framework, but aren't__".

I strive to meet any 4 of the following 5 criteria for all code in these libraries: well-designed, well-documented, well-tested, well-written (concise/beautiful), and high-performance.

The libraries are:

- **[Loyc.Essentials.dll](https://github.com/qwertie/LoycCore/wiki/Loyc.Essentials)**: a library of interfaces, extension methods, and small bits of functionality that are useful in virtually any software project. About half of **Loyc.Essentials** is devoted to collections: collection interfaces, collection adapters, collection extension methods and even a couple of collection implementations (most notably `DList<T>`). The other half includes a variety of things including math, geometry, `Symbol`s, localization, `Pair<A,B>`, "message sinks" and more.
- **[Loyc.Collections.dll](https://github.com/qwertie/LoycCore/wiki/Loyc.Collections)**: a library of sophisticated data structures including [ALists][1], [VLists][2], and my favorite, the hash tree types `Set<T>`, `MSet<T>`, `Map<T>` and `MMap<T>`.
- **[Loyc.Syntax.dll](https://github.com/qwertie/LoycCore/wiki/Loyc.Syntax)**: Contains a parser for [Loyc Expression Syntax (LES)][3], and various interfaces and base classes for Loyc Languages and for users of LLLPG.
- **[Loyc.Utilities.dll](https://github.com/qwertie/LoycCore/wiki/Loyc.Utilities)**: Additional functionality that is either (A) not important enough to be placed in **Loyc.Essentials.dll** or (B) takes **Loyc.Collections.dll** as a dependency.

  [1]: http://www.codeproject.com/Articles/568095/The-List-Trifecta-Part
  [2]: http://www.codeproject.com/Articles/26171/VList-data-structures-in-C
  [3]: https://github.com/qwertie/LoycCore/wiki/Loyc-Expression-Syntax

They are available as a [NuGet package](https://www.nuget.org/packages/LoycCore/).

Dependency tree
---------------

Low-level libraries on top:

         Loyc.Essentials
                |
         Loyc.Collections
                ^   ^      
                |   |      
                |   +----------------+
                |                    |     
                |                    |
          Loyc.Utilities        Loyc.Syntax

Note: the .NET 3.5 versions of these libraries depend on the compatibility library [Theraot.Core.dll](https://github.com/theraot/Theraot).

