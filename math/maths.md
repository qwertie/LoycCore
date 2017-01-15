---
title: Arithmetic in generic code
layout: article
toc: false
date: 10 Dec 2016
---

Ever since .NET got generics over 13 years ago, it has not been easy to do arithmetic in generic code.

    public static T Sum<T>(T a, T b) => a + b; // Error, there's no + operator

This is mainly because the primitive types, like `int`, do not implement any interface to expose operations like "add", "multiply", and "negate". Not even in .NET 4. Or .NET 4.5...

    public static T Sum<T>(T a, T b) where T: IMath<T> // no such thing

In 2004, Rüdiger Klaehn wrote an article about [how it is possible to do arithmetic efficiently in generic classes](https://www.codeproject.com/Articles/8531/Using-generics-for-calculations). His soluition looks something like this:

~~~
class Lists<T,M> where M: IMath<T>, new()
{
    // M should be an empty structure, so we can create an instance "for free"
    private static M math = new M();
    
    public static T Sum(List<T> list) 
    { 
        T sum = default(T); // zero
        for (int i = 0; i < list.Count; i++)
            sum = math.Add(sum, list[i]); 
        return sum;
    } 
}
~~~

Here, `IMath<T>` is a class that has an `Add` method that allows you to add two numbers together. At the time he wrote his article, the .NET JIT did not run this code efficiently, but nowadays the performance of this code is comparable to dedicated code for adding up integers, because the JIT specializes the class for the type parameter `M` (as long as it is a struct) and the call to `Add` can be inlined..

Loyc.Math provides a more diverse set of math operations than Rüdiger originally designed. In conjunction with Loyc.Essentials, which defines the interfaces implemented by Loyc.Math, your generic code can learn everything it might want to know about an unknown numeric type `T` and do any arithmetic it might want to do. For example, Loyc.Math has methods for shifting (Shr(1) is an actual shift-right if the type is an integer, but divides by a power of two if the type is floating point), you can get the next higher value (add one [ULP](https://en.wikipedia.org/wiki/Unit_in_the_last_place)), you can count the number of "1" bits if it's an integer, and you can query properties like the maximum non-infinite value and the number of significant bits.

In Loyc.Math, `MathI` is the "math provider" structure for `int`, `MathD` is the provider for `double`, `MathL` for `long`, `MathU8` for `byte`, and so on. Math structures for fixed-point and even two-dimensional vectors are available, too.

Unfortunately, for high performance, your generic class that does math requires an extra generic parameter `M`, and supplying that parameter tends to be a big hassle for your users. Occasionally they can avoid this hassle using `var`; for example, `Range.Inclusive(1, 10)` returns a range of numbers from 1 to 10, represented by `NumRange<int,MathI>`. One can use `var` to avoid writing down the type:

    var range = Range.Inclusive(1, 10);
    foreach (int num in range)
        Console.WriteLine(num);

Another way for users to avoid mentioning the math provider is to hold the math-enabled object in an interface. For example, `NumRange<int,MathI>` implements `ICollection<int>`, `IReadOnlyList<int>` and `IListSource<int>`. This, however, means that you have to use _interface_ calls to access the object, which might be slow in some circumstances (e.g. if you want to make millions of calls per second). Plus, in the case of `NumRange<int,MathI>`, it is a `struct`, so treating it as an interface like `ICollection<int>` causes boxing.

If your generic class doesn't need to do high-performance arithmetic, then it doesn't need the extra `M` generic parameter when using `Loyc.Math`. Instead it can get an `IMath<T>` object from `Maths<T>`. For example, here's a version of the above code that adds up numbers more slowly, but without the `M` parameter:

~~~
class Lists<T>
{
    private static IMath<T> math = Maths<T>.Math;
    private static IAdditionGroup<T> add = Maths<T>.AdditionGroup;
    
    public static T Sum(List<T> list) 
    { 
        T sum = default(T); // zero
        for (int i = 0; i < list.Count; i++)
            sum = add.Add(sum, list[i]); 
        return sum;
    } 
}
~~~

This example also adds a new field of type `IAdditionGroup<T>`, and uses it for addition instead of `math`. `IMath<T>` provides a complete set of math operations including multiplication, square root, comparisons, and so on, but since all `Sum` needs to do is add things, it is better to use [`IAdditionGroup`](http://ecsharp.net/doc/code/interfaceLoyc_1_1Math_1_1IAdditionGroup.html) so that it works with types that can be added and subtracted but not, for example, sorted. Consider `Vector<T>`, which is a pair of numbers `X` and `Y`; they support addition and subtraction but not much else, so the whole `IMath<Vector<T>>` interface is not available.

**Note:** the members of `Maths<T>` are `null` when the desired math interface is not available.

**Note:** someday this code should work with `Vector<T>`, but as of 2016 `Maths<T>` supports primitive types only.

Example: Geometry
-----------------

Right now, several types of Loyc.Math, such as `Point<T>`, `Vector<T>` and `BoundingBox<T>`, use `Maths<T>` to enable generic arithmetic. Unfortunately, this arithmetic is slow when using those math objects, since there is no separate type parameter `M`. To mitigate this problem, these classes have extension methods to do math for specific types. Basically this is accomplished by repeating code like this many times:

	using T = System.Int32;
	using Point = Point<int>;
	using Vector = Vector<int>;
	using Point3 = Point3<int>;
	using Vector3 = Vector3<int>;
	using LineSegment = LineSegment<int>;
	using System;

	public static partial class PointMath
	{
		public static Vector Add(this Vector a, Vector b) { return new Vector(a.X+b.X, a.Y+b.Y); }
		public static Point  Add(this Point a, Vector b)  { return new Point(a.X+b.X, a.Y+b.Y); }
		public static Point  Add(this Vector a, Point b)  { return new Point(a.X+b.X, a.Y+b.Y); }
		public static Vector Sub(this Vector a, Vector b) { return new Vector(a.X-b.X, a.Y-b.Y); }
		public static Vector Sub(this Point a, Point b)   { return new Vector(a.X-b.X, a.Y-b.Y); }
		public static Point  Sub(this Point a, Vector b)  { return new Point(a.X-b.X, a.Y-b.Y); }
		public static Vector Mul(this Vector p, T factor) { return new Vector(p.X*factor, p.Y*factor); }
		public static Point  Mul(this Point p,  T factor) { return new Point(p.X*factor, p.Y*factor); }
		public static Vector Div(this Vector p, T factor) { return new Vector(p.X/factor, p.Y/factor); }
		public static Point  Div(this Point p, T factor)  { return new Point(p.X/factor, p.Y/factor); }
		...
	}

Each copy of the code changes the meaning of `Point` so that there is fast code for adding `Point<int>`, `Point<double>` and `Point<float>`.

Unfortunately you have to use the ugly syntax `pointA.Sub(pointB)` instead of `pointA - pointB` to get fast arithmetic. There are also operators for `Point<T>` and `Vector<T>`:

	static IMath<T> m = Maths<T>.Math;

	public static Point<T>  operator+(Point<T> a, Vector<T> b) 
	  { return new Point<T>(m.Add(a.X,b.X), m.Add(a.Y,b.Y)); }
	public static Point<T>  operator+(Vector<T> a, Point<T> b)
	  { return new Point<T>(m.Add(a.X,b.X), m.Add(a.Y,b.Y)); }

But these operators are slow, of course. Currently, the C# design team is considering whether to add ["extension everything"](https://github.com/dotnet/roslyn/issues/11159) to C#, including extension operators. Such operators would be very helpful by allowing the _operators_ to use the fast code path and the _methods_ to use the slow path (slow methods based on `Maths<T>.Math` would still be useful to allow generic code to easily do math on points and vectors).

Download
--------

The generic math code described above is in Loyc.Math.dll, published on NuGet. The source code is [here](https://github.com/qwertie/ecsharp/tree/master/Core/Loyc.Math/Math).
