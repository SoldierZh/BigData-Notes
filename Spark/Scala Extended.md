# Scala 拓展知识

## 1. def/val/lazy val

- **def**：使用时执行，每次使用都会执行，进行重新赋值。

```scala
scala> def f = {println("hello"); 1}
f: Int
scala> f
hello
res0: Int = 1
scala> f
hello
res1: Int = 1
```

- **val**：立即执行，只执行一次。

```scala
scala> val a = {println("hello"); 1}
hello
a: Int = 1
scala> a
res2: Int = 1
scala> a
res3: Int = 1
```

- **lazy val** : 延迟执行，使用时再执行，只执行一次。

```scala
scala> lazy val b = {println("hello"); 1}
b: Int = <lazy>
scala> b
hello
res4: Int = 1
scala> b
res5: Int = 1
```

## 2. reduce vs fold

- reduce 的返回值类型必须和列表的类型一致或是其父类，但是fold 没有这种限制，reduce 是 fold 的一种特例。

```scala
List("1", "2", "3").reduceLeft((a, b) => a.toInt + b.toInt)  // error
List("1", "2", "3").foldLeft(0)((a, b) => a.toInt + b.toInt) // ok
```

- reduceLeft / reduceRight 是有序的，reduce 是无序的；foldLeft / foldRight 与 fold 的关系也一样。

