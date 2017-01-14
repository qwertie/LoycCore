---
title: LINQ-to-Lists in Loyc.Essentials
layout: article
toc: false
date: 27 Dec 2016
---

The venerable "LINQ to Objects" ([Enumerable class](https://msdn.microsoft.com/en-us/library/system.linq.enumerable(v=vs.110).aspx)) lets us query and transform any object that implements `IEnumerable<T>` and get another `IEnumerable<T>` in return. But what if our input object is a collection? Shouldn't we be able to query a list with LINQ and get another list in return?

Loyc.Essentials provides a supplement to `System.Linq.Enumerable` called "LINQ-to-lists", formerly LINQ-to-Collections (I changed the name because it does not focus on non-list collections, such as dictionaries). `LinqToLists` has two purposes:

- To **improve the performance** of a few methods of `Enumerable` using the knowledge that an object is a list. For example, `Enumerable.Skip(IEnumerable<T>, int)` has to scan through the specified number of items to skip them, but `LinqToLists.Skip(IList<T> list, int)` skips them without a loop by returning a slice (sublist).
- To **preserve the interface**: if the input is a list, then the output can be a list, too. However, `LinqToLists` only provides methods to preserve the interface in cases where the list size is known immediately, without scanning the list. So there is a `LinqToLists` version of `Select` that returns a list of the same size, but there is no `LinqToLists` version of `Where` because it would be necessary to scan the entire list to determine the output size. Instead, use the normal `.Where()` extension method and call either the `.ToList()` extension method to construct a list from it _greedily_ (immediately), or `.Buffered()` (an extension method of `LinqToLists`) to construct a list from it _lazily_ (on-demand).

**Usage**: add a reference to Loyc.Essentials (available as NuGet package), then add `using Loyc.Collections;` to the top of a source file.

LINQ-to-Lists supports both the traditional `IList<T>` and the new `IReadOnlyList<T>` from .NET 4.5. If you are using the .NET 3.5 version of Loyc.Essentials then `IReadOnlyList<T>` is defined in the compatibility library Theraot.Core.dll; If you are using the .NET 4 version then `IReadOnlyList<T>` is defined by Loyc.Essentials.dll itself.

Plus, LINQ-to-Lists supports ["neg-lists"](http://ecsharp.net/doc/code/interfaceLoyc_1_1Collections_1_1INegListSource.html): lists whose minimum valid index is not necessarily zero.

The extension methods that return lists rely on adapter structures and classes defined in Loyc.Essentials such as `ListSlice<T>`, `SelectList<T,TResult>`, and `ReversedList<T>`. To optimize performance, these types are returned directly instead of returning `IList<T>` or `IReadOnlyList<T>`.

LINQ-to-Lists has following **performance-enhancing methods** (only `IList<T>` overloads are listed here, but generally there are also overloads for `IReadOnlyList<T>` or [`IListSource<T>`](http://ecsharp.net/doc/code/interfaceLoyc_1_1Collections_1_1IListSource.html), and usually `INegListSource<T>`):

- `FirstOrDefault<T>(this IList<T> list, T defaultValue = default(T))`: returns the first item. This has better performance than `Enumerable.FirstOrDefault` since an enumerator object does not have to be created and queried. Plus, you can choose the default value.
- `Last<T>(this IList<T> list)`: returns `list[list.Count - 1]` or throws `EmptySequenceException` if empty. (Note: `Enumerable.Last` does check "does this object implement `IList<T>`? If so, return `list[list.Count - 1]`, but this method avoids the cast, and `Enumerable.Last` does not have a similar optimization for `IReadOnlyList<T>` and will scan the whole list to get the last item.)
- `LastOrDefault<T>(this IList<T> list, T defaultValue = default(T))`: returns `list[list.Count - 1]` or `defaultValue` if empty.
- `Skip<T>(this IList<T> list, int start)`: returns a smaller list without the first `start` items. This is faster since it doesn't scan the items that were skipped. **Note**: this behaves slightly differently from `Enumerable.Skip` since the latter doesn't actually do anything until you start enumerating, whereas this method creates the slice and calculates its size immediately. This method is a synonym for another Loyc.Essentials extension method, `Slice(list, start)`.
- `Count<T>(this IList<T> list)`: returns `list.Count`.

LINQ-to-Lists has the following methods that don't improve performance but **preserve the list interface**:

- `Take<T>(this IList<T> list, int count)`: returns a list whose size is limited to `count` items. If `count` is greater than or equal to the size of the original list, this method returns the original list wrapped in a slice structure ([`ListSlice<T>`](http://ecsharp.net/doc/code/structLoyc_1_1Collections_1_1ListSlice.html)). **Note**: this behaves slightly differently from `Enumerable.Take` since the latter doesn't actually do anything until you start enumerating, whereas this method creates the slice and calculates its size immediately. This method is a synonym for `Slice(list, 0, count)`.
- `TakeNowWhile<T>(this IList<T> list, Func<T, bool> predicate)`: takes elements from the beginning of a list while the specified predicate returns true, and returns a slice that provides access to them. Since the entire (possibly lengthy) operation is done up-front, I decided to call it `TakeNowWhile` instead of just `TakeWhile`.
- `SkipNowWhile<T>(this IList<T> list, Func<T, bool> predicate)`: skips elements from the beginning of a list while the specified predicate returns true, and returns a slice that provides access to the remainder of the list. Since the entire (possibly lengthy) operation is done up-front, I decided to call it `SkipNowWhile` instead of just `SkipWhile`. _Should I have done the same for `Take` and `Skip`?_
- `Select<T, TResult>(this IListSource<T> source, Func<T, TResult> selector)`: returns a "projected" version of the list, so whenever you read the item at index `i`, `selector(source[i])` is called. Unlike other methods of LinqToLists, this method returns a read-only list, but it does implement `IList<T>` as well as `IReadOnlyList<T>`.
- `Reverse<T>(this IList<T> c)`: returns a reversed view of the list (so `Reverse(list)[i]` really means `list[list.Count - 1 - i]`).

Occasionally, preserving the interface yields higher performance; for example, `list.Take(N).Last()` would scan N items when using `Enumerable` but immediately returns `list[Math.Max(list.Count, N) - 1]` when using Loyc.Essentials.

You can find the Enhanced C# source code [here](https://github.com/loycnet/ecsharp/blob/master/Core/Loyc.Essentials/Collections/ExtensionMethods/LinqToLists.ecs). Admittedly, it's not super simple, as it is part of a larger library themed "stuff that should be built into the .NET framework, but isn't". It uses [Enhanced C#](http://ecsharp.net) to generate variations of similar code, and it relies on several adapter types defined [here](https://github.com/loycnet/ecsharp/tree/master/Core/Loyc.Essentials/Collections/Adapters) and helper classes in [here](https://github.com/loycnet/ecsharp/tree/master/Core/Loyc.Essentials/Collections/HelperClasses). Those, in turn, may rely on some of the base classes [here](https://github.com/loycnet/ecsharp/tree/master/Core/Loyc.Essentials/Collections/BaseClasses) and some of the interfaces [here](https://github.com/loycnet/ecsharp/tree/master/Core/Loyc.Essentials/Collections/Interfaces).

### More extension methods! ###

Besides LINQ to Lists, it's also worth noting that Loyc.Essentials has additional extension methods for plain-old `IEnumerable<T>` in [`EnumerableExt`](http://ecsharp.net/doc/code/classLoyc_1_1Collections_1_1EnumerableExt.html).

Help wanted (Jan 2017): I haven't found the time to write unit tests for this stuff in [LoycCore.Tests](https://github.com/loycnet/ecsharp/tree/master/Core/Tests/Essentials).
