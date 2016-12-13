---
layout: page
title: Learn about Loyc.Math
---

### Articles ###

- [Arithmetic in generic code](maths.md)

### Notable math types ###

- [`MathEx`](http://ecsharp.net/doc/code/classLoyc_1_1Math_1_1MathEx.html)
- [`Math128`](http://ecsharp.net/doc/code/classLoyc_1_1Math_1_1Math128.html)
- Fixed-point types: [`FPI8`](http://ecsharp.net/doc/code/structLoyc_1_1Math_1_1FPI8.html) (`int` with 8 fraction bits), [`FPI16`](http://ecsharp.net/doc/code/structLoyc_1_1Math_1_1FPI16.html) (`int` with 16 fraction bits) 
[`FPL32`](http://ecsharp.net/doc/code/structLoyc_1_1Math_1_1FPI16.html) (`long` with 32 fraction bits)
- `Maths<T>`, `MathI`, `MathD`, etc.: see ["Arithmetic in generic code"](maths.md)
- [`Range<T>`](http://ecsharp.net/doc/code/classLoyc_1_1Range.html): supports the EC# range operators `a..b` and `a...b`

### Notable geometry types ###

- [`Point<T>`](http://ecsharp.net/doc/code/structLoyc_1_1Geometry_1_1Point.html) and [`Vector<T>`](http://ecsharp.net/doc/code/structLoyc_1_1Geometry_1_1Vector.html)
- [`Point3<T>`](http://ecsharp.net/doc/code/structLoyc_1_1Geometry_1_1Point3.html) and [`Vector<T>`](http://ecsharp.net/doc/code/structLoyc_1_1Geometry_1_1Vector3.html)
- [`BoundingBox<T>`](http://ecsharp.net/doc/code/classLoyc_1_1Geometry_1_1BoundingBox.html)
- Classes that contain extension methods for each of the above (e.g. optimized for `int` or `double`)

### Interfaces ###

Loyc.Essentials defines most of the interfaces implemented by Loyc.Math, such as [`IPoint<T>`](http://ecsharp.net/doc/code/interfaceLoyc_1_1Geometry_1_1IPoint.html), [`IRectangle<T>`](http://ecsharp.net/doc/code/interfaceLoyc_1_1Geometry_1_1IRectangle.html) and [`IMath<T>`](http://ecsharp.net/doc/code/interfaceLoyc_1_1Math_1_1IMath.html). This way, independent implementations of those interfaces can exist that do not reference Loyc.Math.
