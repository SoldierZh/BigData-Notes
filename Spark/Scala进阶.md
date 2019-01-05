# 6. Scala 进阶

## 6.1 Scaladoc 使用

> Scala api 文档：class、object、function、method、implicit。

- [Sacala在线文档](https://www.scala-lang.org/api/current/)

1. O和C，分别代表某个类的伴生对象和伴生类；
2. 标记为 implicit 的函数，表示隐式转换。

## 6.2 跳出循环语句的3中方法

- 基于 Boolean 类型的控制变量

```scala
var flag = true
var res = 0
var n = 0
// while 循环
while(flag) {
    res += n
    n += 1
    if(n == 5) {
        flag = false
    }
}
// for 循环
flag = true
res = 0
n = 10
for(i <- 0 until 10 if flag) {
    res += i
    if(i == 4) flag = false
}
```

- 在嵌套函数中使用 return

```scala
def add_outer() = {
    var result = 0 
    def add_inner() {
        for(i <- 0 until 10) {
            if(i == 5) {
                return
            }
            result += i
        }
    }
    add_inner()
    result
}
```

- 使用Breaks类的 break

```scala
import scala.util.control.Breaks._
var res = 0
breakable {
    for(i <- 0 until 10) {
        if(i == 5) {
            break
        }
        res += i
    }
} // break 之后会到到这一行
```

## 6.3 多维数组

- `ofDim()`

```scala
// 指定行与列
val multiArray = Array.ofDim[Int](3, 4)
multiArray(0)(0) = 12
// 不规则多维数组
val multiArray = new Array[Array[Int]](3)
multiArray(0) = new Array[Int](5)
multiArray(1) = new Array[Int](7)
multiArray(2) = new Array[Int](9)
multiArray(1)(6) = 11
```

## 6.4  Java List 与 Scala ArrayBuffer 的隐式转换

```scala
import scala.collection.JavaConversions.bufferAsJavaList
import scala.collection.mutable.ArrayBuffer
import java.util.List
val arrayBuffer = ArrayBuffer("A", "B", "C")
def printList(list: List[String]) {
    var i = 0
    while(i < list.size) {
        println(list.get(i))
        i += 1
    }
}
printList(arrayBuffer) // 隐式转换， ArrayBuffer(Scala) ==> List(Java)
import scala.collection.JavaConversions.asScalaBuffer
import scala.collection.mutable.Buffer
val arrayList = new ArrayList[String]
arrayList.add("a")
arrayList.add("b")
val buffer: Buffer[String] = arrayList // 隐式转换， ArrayList(Java) ==> Buffer(Scala)
```

## 6.5 Tuple zip 操作

- 将两个 Array 合并为一个 Array，新的 Array 的每个元素是一个 Tuple 类型

```scala
val students = Array("Leo", "Jack", "Jen") // Array[String]
val scores = Array(80, 90, 100) // Array[Int]
val studentScores = students.zip(scores) // Array[(String, Int)] ， 每个元素是一个 Tuple
```

- 如果Array 的元素是一个 Tuple， 调用 `Array.toMap()` 可将Array转换为 Map

```scala
val map = studentScores.toMap
```

## 6.6  Java Map 与 Scala Map 隐式转换

```scala
import scala.collection.JavaConversions.mapAsScalaMap
val javaScores = new java.util.HashMap[String, Int]()
javaScores.put("A", 80)
javaScores.put("B", 90)
javaScores.put("C", 100)
val scalaScores: scala.collection.mutable.Map[String, Int] = javaScores  // 隐式转换 java Map ==> Scala Map
import scala.collection.JavaConversions.mapAsJavaMap
val javaMap: java.util.Map[String,Int] = scalaScores // 隐式转换 scala Map ==> Java Map
```

## 6.7 扩大内部类作用域的两种方法

- 内部类依赖于外部类的实例对象，而不是依赖外部类

```scala
class Class {
    class Student(val name: String) {}
    val students = new ArrayBuffer[Student]
    def register(name: String) = {
        new Student(name)
    }
}
val c1 = new Class
val leo = c1.register("leo")
c1.students += leo
//
val c2 = new Class
val jack = c2.register("jack")
s1.students += jack // 报错
```

### 6.7.1 伴生对象

- 类似 Java 中的 static 内部类

```scala
object Class {
    class Student(val name: String)
}
class Class {
    val students = new ArrayBuffer[Class.Student]
    def register(name: String) = {
        new Class.Student(name)
    }
}
val c1 = new Class
val leo = c1.register("leo")
c1.students += leo
//
val c2 = new Class
val jack = c2.register("jack")
c1.students += jack // 正确执行
```

### 7.7.2  类型投影

- 使用 `#` 操作符 

```scala
class Class {
    class Student(val name: String)
    val students = new ArrayBuffer[Class#Student]  // 类型投影
    def register(name: String) = {
        new Student(name)
    }
}
val c1 = new Class
val leo = c1.register("leo")
c1.students += leo
//
val c2 = new Class
val jack = c2.register("jack")
c1.students += jack // 正确执行
```

## 6.8 内部类获取外部类的引用

- `outer =>` ： outer 是外部类的引用，可以自定义名称

```scala
class Class(val name: String) { outer =>
    class Student(val name: String) {
        def printMyself = "Hello, I'm " + name + ", outer name is " + outer.name
    }
    def register(name: String) = {
        new Student(name)
    }
}
val c1 = new Class("c1")
val leo = c1.register("leo")
leo.printMyself
```

## 6.9 重写 field 的提前定义

1. 子类的构造函数调用父类的构造函数
2. 父类的构造函数初始化 field
3. **父类的构造函数使用 field 执行其他构造代码，但是此时其他构造代码如果使用了该 field，而且 field 要被子类重写，那么它的 getter 方法会被重写，返回 0 （Int）**
4. 子类的构造函数再执行，重写field
5. 但是此时子类从父类继承的其他构造代码，已经出现了错误

```scala
class Student {
    val classNumber: Int = 10
    val classScores: Array[Int] = new Array[Int](classNumber)
}
class PEStudent extends Student{override val classNumber: Int = 3}
// test
val s1 = new Student
s1.classNumber   // 10
s1.classScores.length  // 10
val s2 = new PEStudent
s2.classNumber // 3
s2.classScores.length // 0
```

- **提前定义** : 在父类构造函数执行前，先执行子类的构造函数中的部分代码。

```scala
class PEStudent extends {
    override val classNumber: Int = 3
} with Student
val s3 = new PEStudent
s3.classNumber // 3
s3.classScores.length // 3
```

## 6.10 Scala 继承层级

- Scala 的 trait 和 class 都默认继承自一些 Scala 的根类，具有一些基础方法，类似Java中的 Object 类；
- Scala 中最顶端的两个 trait 是 **Nothing trait** 和 **Null trait**,  **Null** 唯一的对象就是 **null**；
- **Any class** 类继承了 **Nothing trait**；
- **Anyval trait** 和 **AnyRef class** 继承了 **Any class**；
- **Any class** 类似Java中的**Object**， 其定义了 **isInstanceOf** 和 **asInstanceOf** 等方法，以及 **equals**、**hashCode** 等对象的基本方法；
- **AnyRef class** 增加了一些多线程的方法，比如 **wait**、**notify/notifyAll** 、**synchronized** 等。

## 6.11 对象的相等性

- 重写 `equals` 和 `hashCode` 方法， 类似Java中的操作

## 6.12 I/O操作

### 6.12.1 读取文件

> ``scala.io.Source.fromFile(fileName: String, decode: String)` 方法打开文件， `Source.close`关闭文件。`

- 使用 `source.getLines` 返回迭代器 （**getLines 只能调用一次**）

```scala
import scala.io.Source
val source = Source.fromFile("E://test.txt", "UTF-8")
val lineIterator = source.getLines
for(line <- lineIterator) println(line)
source.close
```

- 将 `Source.getLines` 返回的迭代器转换成数组

```scala
import scala.io.Source
val source = Source.fromFile("E://test.txt", "UTF-8")
val lines = source.getLines.toArray
for(line <- lines) println(line)
source.close
```

- 调用 `Source.mkString` ，返回文本中所有内容

```scala
import scala.io.Source
val source = Source.fromFile("E://test.txt", "UTF-8")
val lines = source.mkString
println(lines)
source.close
```

- 读取文件字符

```scala
import scala.io.Source
val source = Source.fromFile("E://test.txt", "UTF-8")
for(c <- source) print(c)
```

### 6.12.2 读取 URL 的字符（网页）

```scala
import scala.io.Source
val htmlSource = Source.fromURL("https://www.baidu.com", "UTF-8")
for(c <- htmlSource) print(c)
```

### 6.12.3  读取字符串中的字符

```scala
import scala.io.Source
var strSource = Source.fromString("Hello World!")
for(c <- strSource) print(c)
```

### 6.12.4 结合Java IO流 读取任意格式文件

```scala
import java.io._
val file = new File("E://test.txt")
val bytes = new Array[Byte](file.length.toInt)
val fis = new FileInputStream(file)
fis.read(bytes)
fis.close()
```

### 6.12.5 结合 Java IO 流写文件

```scala
import java.io._
val fis = new FileInputStream(new File("E://test.txt"))
val fos = new FileOutputStream(new File("E://test2.txt"))
val buffer = new Array[Byte](fis.available)
fis.read(buffer)
fos.write(buffer, 0, buffer.length)
fis.close()
fos.close()
```

### 6.12.6 递归遍历子目录

```scala
def getSubdirIterator(dir: File): Iterator[java.io.File] = {
    val childDirs = dir.listFiles.filter(_.isDirectory)
    childDirs.toIterator ++ childDirs.toIterator.flatMap(getSubdirIterator _)
}
val iterator = getSubdirIterator(new File("E://"))
for(d <- iterator) println(d)
```

### 6.12.7 序列化与反序列化

```scala
@SerialVersionUID(42L) class Person(val name: String) extends Serializable
val leo = new Person("leo")
import java.io._
// 序列化
val oos = new ObjectOutputStream(new FileOutputStream("E://test.obj"))
oos.writeObject(leo)
oos.close()
// 反序列化
val ois = new ObjectInputStream(new FileInputStream("E://test.obj"))
val leo2 = ois.readObject().asInstanceOf[Person]
```

## 6.13 偏函数

- 偏函数是 **PartialFunction[A, B]**类的一个实例，类似 Map 的功能，这个类有两个方法，**apply()** 、**isDefinedAt()** 
- **apply()** ： 直接调用可以通过函数体内的 case 进行匹配，返回结果
- **isDefinedAt()** ：可以返回一个输入是否跟任何一个case语句匹配，true 或者 false

```scala
val getStudentGrade: PartialFunction[String, Int] = {
    case "Leo" => 90
    case "Jack" => 91
    case "Marry" => 92
}
getStudentGrade("Leo")      	// 90
getStudentGrade.apply("Leo")  //90 这两种调用效果一样
getStudentGrade.isDefinedAt("Leo") // true
```

## 6.14 执行外部命令

- 执行Scala 所在进程之外的程序

```scala
import sys.process._
"ls -al .." !  // linux
"calc" ! // windows
```

## 6.15 正则表达式

```scala
var pattern1 = "[a-z]+"

```

