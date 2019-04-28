---
layout: post
author: Andrey Ilinykh
---

Type variance is an important part of scala type system. If you use scala for application programming, you probably don't pay much 
attention to it. But as soon you start to create a library or just a piece of shared code, type variance becomes crucial.

So, let's see in more details what type variance is and how to use it. Wikipedia tells us 
"variance refers to how subtyping between more complex types relates to subtyping between their components". 
To get some intuition, we will create a simple toy - several traits and case classes:
```scala
scala> sealed trait Shape
defined trait Shape

scala> case class Circle(r: Double) extends Shape
defined class Circle

scala> case class Rectangle(w: Double, h: Double) extends Shape
defined class Rectangle
```

We have ``Shape`` trait here and two implementations- ``Circle`` and ``Rectangle``. Both ``Circle`` and ``Rectangle``
are subtypes of ``Shape``. It means a very simple thing. Everywhere we need ``Shape`` we can use ``Circle`` or ``Rectangle``. 
It is called Liskov criteria.
```scala
scala> val s: Shape = Circle(10) //it is fine
s: Shape = Circle(10.0)

scala> val r:Rectangle = s //doesn't compile
<console>:15: error: type mismatch;
 found   : Shape
 required: Rectangle
       val r:Rectangle = s //doesn't compile
                         ^
```

Which makes perfect sense. Now we create one more type, CSV serializer.
```scala
scala> trait ShapeWriter[T <: Shape] {
     |   def write(s: T): String
     | }
defined trait ShapeWriter

scala> val shapeWriter = new ShapeWriter[Shape] {
     |   override def write(s: Shape) = {
     |     s match {
     |       case Circle(r) => s"Circle,${r.toString}"
     |       case Rectangle(w,h) => s"Rectangle,${w.toString},${h.toString}"
     |     }
     |   }
     | }
shapeWriter: ShapeWriter[Shape] = $anon$1@3b603c85

scala> shapeWriter.write(Circle(20))
res0: String = Circle,20.0

scala> shapeWriter.write(Rectangle(20, 10))
res1: String = Rectangle,20.0,10.0
```
As you can see our ``shapeWriter`` successfully writes both circles and rectangles. So far, so good. 
Now imagine, someone uses our code and he wants to write down rectangles only. We provide CSV writer for both circles and rectangles,
so it is natural to reuse it.
```scala
scala> val rectangeWriter: ShapeWriter[Rectangle] = shapeWriter
<console>:16: error: type mismatch;
 found   : ShapeWriter[Shape]
 required: ShapeWriter[Rectangle]
Note: Shape >: Rectangle, but trait ShapeWriter is invariant in type T.
You may wish to define T as -T instead. (SLS 4.5)
       val rectangeWriter: ShapeWriter[Rectangle] = shapeWriter
                                                    ^
``` 
Oops, it doesn't work. Why? Our writer is capable of writing rectangles. 
The problem is that the scala compiler doesn't know it. For compiler ``ShapeWriter[Shape]`` and ``ShapeWriter[Rectangle]``
are just two different types. The error message tells us "Note: Shape >: Rectangle, but trait ShapeWriter is invariant in type T."
So, now we know what "**invariant** in type" means- there is no any relationship between types which are built from 
some type (Shape) and its subtype (Rectangle). How to fix it? Very easy. Scala compiler gives us a good advise. 
We have to mark type parameter by ``-`` symbol. The right definition of ShapeWriter is
```
trait ShapeWriter[-T <: Shape] {
  def write(s: T): String
}
```
Now we can use our ``shapeWriter`` as ``ShapeWriter[Rectangle]``
```scala
val rectWriter: ShapeWriter[Rectangle] = shapeWriter
rectWriter.write(Rectangle(10, 10))

```
It means a ``ShapeWriter[Shape]`` is a subtype of ``ShapeWriter[Rectangle]``. Such inversed relationship between simple types
(`Shape` and `Rectangle`) and composite types (`ShapeWriter[Shape]` and `ShapeWriter[Rectangle]`) is called **contravariance**.

All functions In scala are contravariant in parameters. This is simplified defenition of `Function1` trait:
```scala
trait Function1[ -T1,  +R] extends AnyRef {
  def apply(v1: T1): R
}
```

Let's move forward and implement CSV reader. A type wich reads comma separated string and constructs a `Shape`. Naive defenition would be:
```scala
scala> trait ShapeReader[T <: Shape] {
     |   def read(str: String): T
     | }
defined trait ShapeReader

scala> val rectReader = new ShapeReader[Rectangle] {
     |   override def read(str: String) = {
     |     val ss = str.split(",")
     |     Rectangle(ss(1).toDouble, ss(2).toDouble)
     |   }
     | }
rectReader: ShapeReader[Rectangle] = $anon$1@3aa836ed

scala> rectReader.read(shapeWriter.write(Rectangle(10, 20)))
res2: Rectangle = Rectangle(10.0,20.0)
```

It reads rectangle successfully. But when we try to use it as a `ShapeReader[Shape]` we get a problem
```scala
scala> val shapeReader: ShapeReader[Shape] = rectReader // it doesn't compile
<console>:15: error: type mismatch;
 found   : ShapeReader[Rectangle]
 required: ShapeReader[Shape]
Note: Rectangle <: Shape, but trait ShapeReader is invariant in type T.
You may wish to define T as +T instead. (SLS 4.5)
       val shapeReader: ShapeReader[Shape] = rectReader // it doesn't compile
                                             ^
```
Again, we have the same problem- because `ShapeReader[T <: Shape]` is invariant in type T, `ShapeReader[Shape]` 
and `ShapeReader[Rectangle]` are different types. We can't assign object of different types. The solution is simple - we just 
make our ShapeReader **covariant**:
```scala
scala> trait ShapeReader[+T <: Shape] {
     |   def read(str: String): T
     | }
defined trait ShapeReader
```
then we ca successfully run `val shapeReader: ShapeReader[Shape] = rectReader`. In other words `ShapeReader[Rectangle]` is a subtype
of `ShapeReader[Shape]`.

Following table summarizes the result. It describes relationship between types `SomeType[T]` and `SomeType[S]` when `S` is subtype of `T`

|Variance|Syntax|Description
|--------|------|------
|Covariant|SomeType[+T]|SomeType[S] is subtype of SomeType[T]
|Contravariant|SomeType[-T]|SomeType[T] is subtype of SomeType[S]
|Invariant|SomeType[T]| there is no any relationship

You may ask a question: Why do we need such overcomplicated things. Why not use the same variance everywhere? 
It turns out, the same variance makes the code less safe or makes impossible creation some very usefull classes. 
Let's take some examples. 
Arrays in scala are invariant. The definition looks like `final class Array[T]`.  
Following code doesn't compile in scala
```scala
scala> val rectArray = Array(Rectangle(10, 20))
rectArray: Array[Rectangle] = Array(Rectangle(10.0,20.0))

scala> val shapeArray: Array[Shape] = rectArray
<console>:14: error: type mismatch;
 found   : Array[Rectangle]
 required: Array[Shape]
Note: Rectangle <: Shape, but class Array is invariant in type T.
You may wish to investigate a wildcard type such as `_ <: Shape`. (SLS 3.2.10)
       val shapeArray: Array[Shape] = rectArray
                                      ^
```

But it works in Java because Java arrays are covariant. Imagine Arrays are covariant in scala as well. 
Then very easy we can get a problem. As `shapeArray` is array of shapes, 
we can put `Circle` there: 
```
shapeArray(0) = Circle(5)
```
but because both shapeArray and rectArray point to the same object now, next line will crash our application
```
val area = rectArray(0).w * rectArray(0).h // runtime exception here
```

This code is correct in Java but not in scala.

Another possible way is to have everything invariant. But it brings problems as well. Just one example.

Scala class `Option` is covariant, it is defined as `final class Option[+T]`. Then we have `case class Some[T](value: T) extends Option[T]`
and `object None extends Option[Nothing]`
because `Nothing` is subtype of every other type we can assign it to any `Option`
```scala
scala> val r:Option[Circle]= None
r: Option[Circle] = None
```
But if we have all types invariant then `Option[Circle]` and `Option[Nothing]` are just different types and assignment is not possible.



