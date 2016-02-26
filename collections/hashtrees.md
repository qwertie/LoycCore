---
layout: article
title: A flexible family of hash trees
toc: true
---
You can make "sets" in the .NET framework with the standard `HashSet<T>` class, but it is a little inconvenient compared to using ordinary numbers.

To demonstrate this, consider how easy it is to pass one number to a method:

    Method(42.0);

Passing a set of numbers isn't too difficult, though it's clearly not as smooth:

    Method(new HashSet<double> { 0.25, 0.5, 1.0, 2.0 });

There are some significant difficulties though. First, if you want to pass the same set to multiple functions, we might have a problem...

    var set = new HashSet<double> { 0.25, 0.5, 1.0, 2.0 };
    Method1(set);
    Method2(set);

What's wrong with this? Well, a `HashSet` is mutable. That means `Method1` or `Method2` is free to modify our set. If that's what we _want_, great. But if not, we have to check the documentation of `Method1` and `Method2` to see if they might modify the set, and if they do, we have to make a copy of the set. This is not convenient to do, either; since there is no clone method, we have to call the copy constructor instead, which means repeating the data type with each copy:

    HashSet<double> set = new HashSet<double> { 0.25, 0.5, 1.0, 2.0 };
    Method1(new HashSet<double>(set));
    Method2(new HashSet<double>(set));

And this is a waste of time and memory in case the methods _do not_ modify the sets.

Introducing Loyc.Collections's set types
----------------------------------------

I wrote a different kind of engine for storing sets, modeled after functional languages. First I will show you why these sets are easier to use, and then I will demonstrate how Loyc sets use memory more efficiently.

First, let's see Loyc's immutable `Set` type in action:

    Set<double> set = new MSet<double> { 0.25, 0.5, 1.0, 2.0 }.AsImmutable();
    Method1(set);
    Method2(set);

<div class='sidebox'><b>Note</b>: currently `Set` is a value type, so happily `AsImmutable` does not allocate any memory, but I'm considering whether it should be a reference type instead, since the other three types (`MSet`, `Map` and `MMap`) are already reference types. By the way, I may add a `params` constructor in the future so you can write `new Set<double>(0.25, 0.5, 1.0, 2.0)`, avoiding the need to call `AsImmutable()`. Alternately, I might define a new `static class Set` so you can write simply `Set.New(0.25, 0.5, 1.0, 2.0)`, but in the meantime, you could always write that class yourself if you need an easy way to make sets.</div>
Here, we start with `MSet<double> {...}` in order to use the `{...}` notation (C# calls the `Add()` method of `MSet`, which does not exist in `Set`) and then we call `AsImmutable` to create an immutable version of the set. Since `Set` is immutable, we know—without consulting any documentation—that `Method1` and `Method2` won't modify our set.

Now let's consider what we have to do if we want to pass the combination of two sets, or exclude one set from another:

    var set1 = new HashSet<double> { 0.25, 0.5, 1.0, 2.0 };
    var set2 = new HashSet<double> { 2, 3, 5, 7, 11 };
    
    // pass the union (combined set) to Method1, and the difference to Method2
    var combined = new HashSet<double>(set1);
    combined.UnionWith(set2);
    Method1(new HashSet<double>(combined));
    var difference = new HashSet<double>(set1);
    difference.ExceptWith(set2);
    Method2(new HashSet<double>(difference));

Surely, though, this could be much more convenient. And with Loyc's `Set` type, it is!

    var set1 = new MSet<double> { 0.25, 0.5, 1.0, 2.0 }.AsImmutable();
    var set2 = new MSet<double> { 2, 3, 5, 7, 11 }.AsImmutable();
    
    // pass the union (combined set) to Method1, and the difference to Method2
    Method1(set1 | set2);
    Method2(set1 - set2);

You can use the `set1 | set2` to merge sets, `set1 & set2` to compute an intersection, `set1 ^ set2` to get the exclusive or (items that are in one set but not the other), and, of course, `-` computes the difference between two sets. Finally, you and use `+` and `-` to add or remove a single item (e.g. `set2 + 13.0` creates a new set that contains `13.0`, leaving the original `set2` unchanged.)

Loyc.Collections.dll includes 4 "main" set types:

- [`Set<T>`](http://loyc.net/doc/code/structLoyc_1_1Collections_1_1Set_3_01T_01_4.html) is an immutable set with several operators for combining sets, as you've seen. These operators also have ordinary method forms: you can say `set1.Union(set2)` rather than `set1 | set2` if that seems clearer to you. As with `HashSet`, there are also several boolean methods like `IsSubsetOf(s)` and `SetEquals(s)`. `Set<T>` implements the `ISetImm<T>` interface, which is the immutabe equivalent to the standard `ISet<T>` interface.
- [`MSet<T>`](http://loyc.net/doc/code/classLoyc_1_1Collections_1_1MSet_3_01T_01_4.html) is a mutable set, much like `HashSet<T>`, but more convenient since it has all the same operators as `Set<T>`. Plus, you can cast from `MSet` to `Set` or vice versa in constant time (instantly).
- [`Map<Key,Value>`](http://loyc.net/doc/code/classLoyc_1_1Collections_1_1Map_3_01K_00_01V_01_4.html) is an immutable map of key-value pairs, with convenient methods to create derived maps, e.g. `map.With(key, value)` creates a map with an additional item, while `map.Union(map2)` adds all the items from `map2` that are not already present in `map`.
- [`MMap<Key,Value>`](http://loyc.net/doc/code/classLoyc_1_1Collections_1_1MMap_3_01K_00_01V_01_4.html) is a mutable map of key-value pairs. It works exactly like `Dictonary<Key,Value>` except that it has the same extra capabilities as `Map` does. Plus, you can cast from `MSet` to `Set` or vice versa in constant time (instantly).

Finally, I wrote an extra class `InvertibleSet<T>` which is similar to `Set<T>`, except that it represents a set that may be inverted. For example, you could store a set of numbers `{2, 3, 5, 7}`, or you could have an _inverted_ set of "all numbers except `{2, 3, 5, 7}`".

As with all of my collections, the set classes have a richer set of functionality than the standard .NET framework classes. For example, there is an `AddOrFind(ref item, bool replaceIfPresent)` method in `MSet` and `MMap` that allows you to add a new item and/or retrieve a matching item that already exists. And `MMap` has a method `GetAndRemove(K key, ref V valueRemoved)` that allows you to delete a pair while also retrieving the value associated with that pair.

How they work
-------------

You can cast from `MSet` to `Set`, or vice versa, in O(1) time (instantly), while adding a new item to a set is an O(log N) operation for a set of size N, which means that adding and removing items gets slightly slower for large sets. How does it work?

In fact, all of these "set" and "map" classes are built on a single engine, a value type called `InternalSet<T>`. `InternalSet` implements a hash tree: a tree whose shape depends on the hashcodes of the items in the tree. There are 16 slots (4 bits) for items at each level of the tree, and the tree is at most 8 levels deep, corresponding to the 8*4=32 bits of a hashcode. More specifically, different levels of the tree depend on different parts of the hashcode. The root of the tree places items according to bits 0 to 3 of the hashcode (that's 4 bits), the second level places items according to bits 4 to 7, and so on.

Don't worry too much if you don't understand what I'm saying very well: this is just a summary, it's not meant to be a complete explanation, and teaching advanced computer science concepts is outside the scope of this article. You can use these classes without knowing how they work, but if you already understand how trees and hashtables work, then hopefully as you read on, you'll get a rough idea of what's going on. And if not, you can either (A) give up now, or (B) study [trees](https://en.wikipedia.org/wiki/Tree_(data_structure)), [tries](https://en.wikipedia.org/wiki/Trie) and [hashtables](https://en.wikipedia.org/wiki/Hash_table) first, then come back here, because `InternalSet` is basically a cross between a tree/trie and a hashtable.

Now, in order to "instantly" convert a mutable set into an immutable one, the tree has a "frozen" flag which simply has to be set on the root node (child nodes are also marked frozen incrementally, as the tree is modified; the rest of the tree is marked frozen immediately because this would slow down the conversion process). When an "immutable" tree like `Set<int>` is converted to `MSet<int>`, the root node is already marked frozen, but `MSet<int>` doesn't mind; it has copy-on-write behavior, so that parts of the tree are automatically "thawed" whenever you change the set.

To see how thawing (i.e. converting from frozen to unfrozen) works, let's suppose you start with a tree with the following 12 nodes:

                                 _root**
    ** IsFrozen=true               |
                                   |
          +---------+---------+----+----+---------+---------+
          |         |         |         |         |         |
         0x2       0x3       0x6       0x7       0x9       0xF
                    |                   |         |
                 +--+--+                |      +--+--+
                 |     |                |      |     |
               0x13   0x73             0x57  0x09   0x59

Each node contains up to 16 nodes and/or children, and the sum of nodes and children never exceeds 16. Please note that the diagram below only represents the nodes themselves; the items in this set are not shown, but a set with this many nodes probably contains about 100 items (TODO: measure this).

Since the root node is frozen, all the others are also frozen by implication, even if (as shown here) their own `IsFrozen` flag is not set. Now, in order to add one new item to the tree, some of the nodes will have to be duplicated in order to "unfreeze" them. 

Concretely, suppose that an item is added to node `0x9` (e.g. something whose hashcode ends in `0x39` could go in this node). Before the new item can be placed in node `0x9`, that node must be thawed. To thaw it, an unfrozen copy is made, leaving the original untouched. The copy is not frozen (`IsFrozen==false`), but it does point to the same frozen children. So the new `0x9` node will point to the same two children (`0x09` and `0x59`), and a for-loop sets the `IsFrozen` flag of each child of the duplicated node. Then, the new item is added to the copy of node 0x9. Next, the `_root` is also unfrozen by making a copy of it with `IsFrozen=false`. Again, a for-loop sets the `IsFrozen` flag of each frozen child, and then child slot `0x9` in the root is replaced with the new copy of 0x9 (which has the new item).

This concludes the thawing process. At this point, just two nodes are actually unfrozen, and the modified tree looks like this:

    ! Unfrozen copy              _root!
    ** IsFrozen=true               |
                                   |
          +---------+---------+----+----+---------+---------+
          |         |         |         |         |         |
         0x2**     0x3**     0x6**     0x7**     0x9!      0xF**
                    |                   |         |
                 +--+--+                |      +--+--+
                 |     |                |      |     |
               0x13   0x73             0x57  0x09** 0x59**

There are 12 nodes here and 2 have been copied. The other 10 nodes are still shared between the modified tree and the clone. Next, if you add an item to node `0x6`, only that one node has to be thawed; the root has already been thawed and there is no need to make another copy of it. Due to the 
random nature of hashcodes, it is probable that as you modify the set after cloning it, it is typical for each modification to require approximately one node to be thawed, until the majority of the nodes have been thawed.

`InternalSet` does not thaw unnecessarily. If you try to remove an item that is not present, none of the tree will be thawed. If you add an item that is already present in a frozen node (and you do not ask for replacement), that node will not be thawed. `Contains()` and `Find()` never cause thawing.

For a more complete explanation of how `InternalSet` works, please read the 
[documentation of `InternalSet<T>`](http://loyc.net/doc/code/structLoyc_1_1Collections_1_1Impl_1_1InternalSet_3_01T_01_4.html).

Download today!
---------------

All the set and map types showcased here are part of Loyc.Collections.dll in the Loyc Core package, available on NuGet. Learn more at [core.loyc.net](http://core.loyc.net).

---------------

**TODO**: Add performance information to this page. Summary: `Set<T>` and `MSet<T>` tend to be a bit slower than `HashSet<T>`, but in some cases they use far less memory if your program takes advantage of the persistent storage mechanism (keeping around different versions of the same set, taking advantage of the copy-on-write behavior of the tree.)

---------------