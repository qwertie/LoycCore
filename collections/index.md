---
layout: page
title: Learn about Loyc.Collections
---

### Articles about Loyc.Collections ###

- [The List Trifecta: Part 1](alists-part1.html): The `AList<T>`
- [The List Trifecta: Part 2](alists-part2.html): The `BList<T>`, `BDictionary<K,V>` and `BMultiMap<K,V>`
- [The List Trifecta: Part 3](alists-part3.html): Benchmarks and the `SparseAList<T>`
- [`InternalList<T>`: the low-level `List<T>`](internal-list.html)
- [`DList<T>`: the circular `List<T>`](dlist.html)
- [VList Data Structures in C#](http://www.codeproject.com/Articles/26171/VList-data-structures-in-C)
- [CPTrie: A sorted data structure for .NET](http://www.codeproject.com/Articles/61230/CPTrie-A-sorted-data-structure-for-NET)
- [A flexible family of hash trees](hashtrees.html): `Set<T>`, `MSet<T>`, `Map<T>`, `MMap<T>`, and `InvertibleSet<T>`

**Note**: most of these types are defined in Loyc.Collections.dll. However, `InternalList<T>` and `DList<T>` are defined in Loyc.Essentials so they can be more widely available, while `CPTrie` is defined in Loyc.Utilities because it is rarely used. Regardless of which assembly contains them, all collections are in the Loyc.Collections namespace.

### Collection interfaces ###

Loyc collections use the standard .NET interfaces such as `IEnumerable<T>` and `IList<T>`, but [many additional interfaces](http://loyc.net/2014/using-loycessentials-collection.html) are defined in Loyc.Essentials.dll (which is referenced by Loyc.Collections.dll).
