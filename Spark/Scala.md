# 1. Scala 基本语法

## 1.1 环境搭建

- [Scala官网](https://www.scala-lang.org/download/)  下载 [scala-2.12.8.msi](https://downloads.lightbend.com/scala/2.12.8/scala-2.12.8.msi) 

- 安装 Scala 并配置 环境变量 ， 前提需要安装 JDK 1.8

- 进入CMD，运行 scala 命令，进入scala命令行

## 1.2 Scala 解释器

- **REPL**： Read -> Evaluation -> Print ->Loop
- Scala 底层是 JVM， 所有Scala源码需要编译成 class文件，然后才可以执行。

## 1.3 声明变量

- **val** ：声明常量， `val a = 1`
- **var** ：声明变量， `var b = 2` 
- 指定类型： `var b:String = "hello"`
- 声明多个变量： `var name1,name2:String = null`

## 1.4 数据类型与操作符

- 基本数据类型：Byte、Char、Short、Int、Long、Float、Double、Boolean；

- 增强版基本数据类型：StringOps、RichInt、RichDouble、RichChar，提供了更多功能函数， 基本数据类型在必要时会隐式转换为其对应的增强版类型；

- 基本操作符：与Java的不同点在于，没有++、-- 运算符； 1+1与 1.+(1) 效果相同，相当于调用函数。

## 1.5 代码块

- 多行语句：每个代码块的最后一个语句的值会作为代码块的返回值
```scala
val a = {
    print("Hello")
    2 *3
}
var b = {print("Hello"); 2*3}
```

## 1.6 函数调用与apply函数

- 函数调用方式与Java相似，不同点在于，Scala中如果函数没有参数，则调用时可以不写括号；
- **apply** 函数：自定义类中可以重写apply函数，“类名()” 的形式，其实就是 “类名.apply()” 的缩写，用来创建一个对象，而不使用 “new 类名()” 的方式创建对象 。

## 1.7 条件控制 if else

-  带返回值的语句： `val type = if(age > 18) 1 else 0`， Scala会自动推断返回类型，取 if 和 else 的公共父类型，如 `var type = if(age > 18) 1 else "no"` ， type的类型为 Any，如果没有 else语句，则相当于 `else ()` ；
-  不带返回值的语句： `if(age > 18) type = 1 else type = 0` ;
-  if 子句包含多行：需要使用 `{ }` ，或者进入 `:paste` 编辑之后，再 `ctrl+D`的方式执行。

## 1.8  输入输出

- **readLine()** ： `val name = readLine("please show your name :\n")`, `var name = readLine()`,  或者 `var name = io.StdIn.readLine()`;
- **readInt()** : `val age = readInt()`;
- **print()**： `print("Hello World")`，相当于 Java 的 `System.out.print()`；
- **println()**：带换行的输出，相当于 Java 的 `System.out.println()`；
- **printf()**：格式化输出， `printf("your age is %d" , age)`，相当于 Java 的 `System.out.printf()`；

## 1.9 循环语句 

- **while do** ： 使用与 Java 相同 
- **for**：`var n = 10; for(i <- 1 to n) println(i)` 会输出 10，  `var n = 10; for(i <- 1 until n) println(i)` 不会输出10，会忽略下边界， `for (i <- "Hello") println(i)` 会输出每一个字符；
- 跳出循环语句：Scala 中没有 break 语句， 可以使用 boolean 类型变量、return 或者 Breaks 的break和 breakable；
```scala
import scala.util.control.Breaks._
breakable {
    var n = 10
    for(c <- "Hello World") {
        if(n == 5) break;
        print(c)
        n -= 1
    }
}
```
- 多重 for 循环： 九九乘法表
```scala
for(i <- 1 to 9; j <- 1 to 9) {
    if(j == 9){
        print(i * j)
    } else {
        print(i * j + " ")
    }
}
```
- if 守卫：取偶数
```scala
for(i <- 1 to 100 if i % 2 == 0) println(i)
```
- for 推导式：构造集合
```scala
for(i <- 1 to 10) yield i
```

## 1.10 函数入门

- 定义函数（不需要 `return` 语句返回，代码块的最后一行就是函数的返回值）
```scala
def sayHello(name: String, age: Int): Int = {
    if(age >= 18) {
        printf("Hi, %s, you are a big boy", age)
        age
    } else {
        printf("Hi, %s, you are a child", age)
    	age
    }
}
```
- 函数调用
```scala
sayHello("Li", 10)
```
```scala
sayHello(age = 10, name = "Li")
```
- 递归函数
```scala
def fab(n: Int): Int = {
    if(n <= 1) 1
    else fab(n - 1) + fab(n - 2)
}
```
- 默认参数（参数列表最右边）
```scala
def sayHello(firstName: String, middleName: String, lastName: String = "Bob") = {
    firstName + middleName + lastName
}
```
- 变长参数 （**使用序列调用变长参数函数， `sum(1 to 5: _*)`**）
```scala
def sum(nums: Int*) = {
    var result = 0
    for(num <- nums) {
        result += num
    }
    result
}

sum(1, 2, 3, 4, 5)
sum(1 to 5: _*)

// 递归方式， nums其实是 Range.Inclusive 类型
def sum2(nums: Int*): Int = {
    if(nums.length == 0) 0
    else nums.head + sum2(nums.tail: _*)
}
```
- 过程 ： 返回值为 Unit 的函数，过程通常用于不需要返回值的函数
```scala
def sayHello(name: String) = "Hello, " + name
def sayHello(name: String) {print("Hello, " + name); "Hello, " + name}
def sayHello(name: String): Unit = "Hello, " + name
```
- **lazy** 值：如果将一个变量声明为 lazy，则只有在第一次使用该值时，变量对应的表达式才会发生计算； 必须是 val 型常量值
```scala
import scala.io.Source._
lazy val lines = fromFile("E://log.txt").mkString
```

## 1.11 异常捕获和处理

- 与Java类似
```scala
try {
	//...
    throw new  IllegalArgumentException("测试异常")
} catch {
    case e1: IllegalArgumentException => print("IllegalArgumentException")
    case e2: IOException => print("IOException")
} finally {
    print("release resources!")
}
```

## 1.12 数组

- **Array**  (对应 Java 中的数组，长度不可变的数组)
```scala
val a = new Array[Int](10)  // [Int] 表示 泛型,每个元素会初始化为 0 
a(0)     // 访问数组元素
a(0) = 1 // 设置数组元素
a.length // 数组长度

val a = Array("Hello", "World") // 元素类型自动推断，等同于 val a = Array.apply("Hello", "World")
```
- **ArrayBuffer** (对应 Java 中的 ArrayList， 长度可变的数组)
```scala
import scala.collection.mutable.ArrayBuffer

val b = ArrayBuffer[Int]() // 创建

b += 1  			//  += 添加一个元素
b += (2,3,4,5) 		// += 添加多个元素
b ++= Array(6,7,8,9)// ++= 添加其他集合中的所有元素

b.insert(5, 6)		// 在 index = 5 的位置插入元素 6
b.insert(5,7,8,9,10) // 在 index = 5 的位置插入多个元素

b.remove(1)        // 移除 index=1 的元素
b.remove(1，3) 	  // 移除 index=1 及其之后的 3个元素
b.trimEnd(5)  	// 可以从尾部截断指定个数的元素

b.toArray    // ArrayBuffer 转换成 Array  
a.toBuffer   // Array 转换成 ArrayBuffer
```
- 遍历数组
```scala
for(i <- 0 until b.length) print(b(i) + " ")
for(i <- 0 until (b.length, 2) print(b(i) + " ") // 2 是步长，隔一个元素输出
for(i <- (0, until b.length).reverse) print(b(i) + " ") // 倒序输出
for(e <- b) print(b(i) + " ") // 增强 for循环
```
- 数组常见操作
```scala
val a = Array(1,2,3,4,5)
var sum = a.sum		// 数组求和
val max = a.max		// 最大值
scala.util.Sorting.quickSort(a) // 排序
// 将数组转换为字符串
a.mkString
a.mkString(",")
a.mkString("<", ",", ">")
a.toString
b.toString
```
- 使用 **yield** 转换数组
```scala
val a = Array(1 to 10: _*)
val a2 = for(ele <- a) yield ele * ele  // 将每个元素转为其平方
val a3 = for(ele <- a if ele % 2 == 0) yield ele * ele  // 结合 if 守卫语法
```
-  使用**函数式编程**转换数组
```scala
a.filter(_ % 2 == 0).map(2 * _)  // _ 表示占位符，代表每个元素
a.filter{_ % 2 == 0} map {2 * _} // 很少使用
```

## 1.13 Map

- 创建 Map
```scala
val m = Map("L" -> 10, "J" -> 20) // 创建不可变 Map
// m("L") = 11  报错
val m2 = scala.collection.mutable.Map("L" -> 10, "J" -> 20) // 创建可变的Map
m2("L") = 11
val m3 = Map(("L", 10), ("J", 20)) //另外一种创建形式
val m4 = new scala.collection.mutable.HashMap[String, Int] // 创建一个空的HashMap
m4("A") = 100
```
- 访问 Map 元素
```scala
// 获取指定key对应的value, 如果 key 不存在，会报错
m1("L")
m1.contains("L") // 判断是否存在key
m1.getOrElse("L", 0) // 如果不存在则返回指定值
```
- 修改 Map 元素
```scala
// 修改可变 Map
m2("L") = 11  // 修改值
m2 += ("M" -> 30, "T" -> 40) // += 增加元素
m2 -= "M" // -= 移除元素
// 修改不可变 Map
val m = m1 + ("M" -> 30)
val m = m1 - "M"
```
- 遍历 Map 元素
```scala
// 遍历 entrySet
for((key, value) <- m) println(key + " " + value)
// 遍历 key
for(key <- m.keySet) println(key)
// 遍历 value
for(value <- m.values) println(value)
// 生成新 map， 反转 key 和 value
for((key, value) <- m) yield (value, key)
```
- **SortedMap**：可自动根据key进行排序
```scala
val m = new scala.collection.mutable.SortedMap[String, Int]
```
- **LinkedHashMap**： 可以记住插入 entry 的顺序
```scala
val m = new scala.collection.mutable.LinkedHashMap[String, Int]
```

## 1.14 Tuple

- 简单 Tuple
```scala
val t = ("K", 10) // 创建
t._1  // 访问
t._2
```
- zip 操作
```scala
val names = Array("A", "B", "C")
val ages = Array(1, 2, 3)
val nameAges = names.zip(ages)
for((name, age) <- nameAges) println(name + ":" +age)
// nameAges 的类型为 Array[(String, Int)]
```

# 2. Scala 面向对象编程

> Scala 既可以面向对象编程，也可以面向过程编程（函数可以独立存在）。

## 2.1 基本概念

### 2.1.1 定义类 class

```scala
class HelloWorld {
    private var name = "L"
    def sayHello() = {
        println("Hello, " + name)
    }  // 调用此函数时，可以加 () ， 也可以不加 ()
    def getName = name  // 调用此函数时，不能加 ()
}

val hw = new HelloWorld()
hw.sayHello()
hw.sayHello
print(hw.getName)
```

### 2.1.2 getter 与 setter

> - 定义不带 **private** 的 **var field**，此时**scala**生成的面向 JVM 的类时，会定义为 **private** 的name字段，并提供对应的 **public** 的 **getter**与 **setter** 方法 ;
> - 如果使用 **private** 修饰 **field**， 则生成的**getter**和**setter**也是**private**的;
> - 如果定义 **val filed** ， 则只会生成 **getter** 方法；
> - 如果不希望生成 **setter** 和 **getter** 方法，则将 **field** 声明为 **private[this]**；
> - 调用 **setter** 和 **getter** 方法，则为：**`name` **和 **`name_=()`  **

```scala
class Student {
    var name = "Defaul"  // 生成 public 的 setter 和 getter 方法
    val age = 10 // 生成 public 的 getter 方法
    private var gender = 1  // 生成 private 的setter 和 getter 方法
}

val student = new Student
print(student.name)
student.name_=("L")
```

### 2.1.3 自定义 getter 与 setter 

```scala
class Student {
    private var myName = "D"
    def name = "your name is " + myName
    def name_=(newValue: String) { // _= 前后不能有空格！！！
        print("you cannot edit your name")
    }
}
```



## 2.2 object

### 2.2.1 object 基本概念

> - scala 的 class 没有 static 域， object 的功能类似Java 的 Class 对象；
> - object 会将内部不在 def 方法中的代码放置到自己的 constructor 中，第一次调用 object 的方法时，会执行这个 constructor， 并且只会执行一次；  
> - object 通常用于作为单例模式的实现，或者放 class 的静态成员，比如工具方法；

```scala
object Person {
    private var eyeNum = 2
    println("this is a person object!") // 只会执行一次
    def getEyeNum = eyeNum
}

Person.getEyeNum
```

### 2.2.2 伴生对象

> - **object** 与一个 **class** 同名，则这个 **object** 就是 这个 **class** 的伴生类；
> - 伴生类和伴生对象必须存放在同一个 .scala 文件中；
> - 伴生类和伴生对象，可以相互访问 **private field**

```scala
object Person {
    private val eyeNum = 2
    def getEyeNum = eyeNum
}

class Person(val name: String, val age: Int) {
    def sayHello = println(name + ": " + age + ": " + Person.eyeNum)
}
```

### 2.2.3  object 继承抽象类

> object 可以继承抽象类，并且覆盖抽象类中的方法

```scala
abstract class Hello(var message: String) {
    def sayHello(name: String): Unit
}

object HelloImpl extends Hello("hello") {
    override def sayHello(name: String) = {
        println(message + "," + name)
    }
}
```

### 2.2.4 apply 方法

> - 通常在伴生对象中实现 **apply** 方法，并在其中实现构造伴生类的对象的功能；
> - 创建伴生类的对象时，通常不会使用**new Class** 的方式，而是使用 **Class()** 方法，隐式调用伴生对象的 **apply** 方法，这样会让对象创建更加简洁

```scala
val a = Array(1,2,3,4) //其实是调用了 object Array 的 apply 方法

// 自定义 apply 方法
class Person(val name: String)
object Person {
    def apply(name: String) = new Person(name)
}
```

### 2.2.5 main 方法

- main 方法必须定义在 object 中
```scala
object HelloWorld {
    def main(args: Array[String]) {
        println("Hello World!")
    }
}
```
- 将代码放到 **.scala** 文件中，然后使用 **scalac** 编译，再用 **scala** 执行，类似 java 操作

### 2.2.6 使用 object 实现枚举功能

- Scala 没有直接提供 Java 中的 Enum 特性，需要通过 object 继承 Enumeration 类， 并且调用 Value 方法来初始化枚举值
```scala
object Season extends Enumeration {
    val SPRING, SUMMER, AUTUMN, WINDTER = Value
}
```
- 可以通过 Value 传入枚举值的**id**和**name**, 通过**id**和 **toString**
```scala
object Season extends Enumeration {
    val SPRING = Value(0, "spring")
    val SUMMER = Value(1, "summer")
    val AUTUMN = Value(2, "autumn")
    val WINTER = Value(3, "winter")
}
Season(0)
Season.withName("spring")
```



## 2.3 继承

- **extends** 子类继承父类的关键字；
- **final** 关键字和 Java 一样， final class 不可继承， final field 不可修改，final method 不可覆盖；
- **override** ：子类要覆盖一个父类的非抽象方法，则必须使用此关键字；子类覆盖父类的 field, 也需要使用此关键字；
- **super**：在子类中调用父类的同名方法；
- 子类可以覆盖父类的 **val field**

### 2.3.1 isInstanceOf 和 asInstanceOf

- 将父类对象强转为子类对象，需要使用 **isInstanceOf[类]** 关键字判断，然后用 **asInstanceOf[类]** 进行强转

```scala
class Person
class Student extends Person
val p: Person = new Student
var s: Student = null
if(p.isInstaceOf[Student]) s = p.asInstanceOf[Student]
```

### 2.3.2 getClass 和 classOf

- **isInstanceOf** 只能判断出对象是否是指定类以及其子类的对象，而不能精确判断出对象就是指定类的对象；
- **getClass** 和 **classOf** 可以精确判断对象就是指定类的对象
- **对象.getClass**可以精确获取对象的类，**classOf[类]** 可以精确获取类，然后使用 **==** 进行比较

```scala
class Peron
class Student extends Person
val p: Person = new Student
p.isInstanceOf[Person]  // true
p.isInstanceOf[Student] // true
p.getClass == classOf[Person] // false
p.getClass == classOf[Student] // true 	
```

### 2.3.3 match 模式匹配进行类型判断

- 使用模式匹配，与 **isInstanceOf** 一样，也是判断主要是该类以及该类的子类的对象即可，不是精确判断；
- 在实际开发中，spark源码中，大量使用了模式匹配进行类型的判断。

```scala
class Person
class Student extends Person
val p: Person = new Student

p match {
    case p1: Peron => println("it's Person's object")
    case p2 => println("unknown type")
}
```

### 2.3.4 constructor 

- 每个类都有一个主 **constructor** 和任意多个辅助 **constructor**， 而每个辅助 **constructor** 的第一行都必须是调用其它辅助**constructor**或者主**constructor**；

- 子类的辅助**constructor**无法直接调用父类的 **constructor**， 只能在子类的主 **constructor** 中调用父类的 **constructor** 

- **注意**：如果是父类中接收的参数，子类中接收时就不需要使用val或者var来修饰，否则会被认为是子类覆盖父类的 fields

```scala
class Person(val name: String, val age: Int)
class Student(name: String, age: Int, var score: Double) extends Person(name, age) {
    def this(name: String) {
        this(name, 0, 0)
    }
    def this(age: Int) {
        this("L", age, 0)
    }
}
```

### 2.3.5 匿名内部类

- 定义一个没有名称的子类

```scala
class Person(protected val name: String) {
    def sayHello = "Hello, " + name
}
val p = new Person("bob") {
    override  def sayHello = "Hi, " + name
}
def greeting(p:Person {def sayHello: String}) {
    println(p.sayHello)
}
```

### 2.3.6 抽象类

- **abstract** ：只有有个方法是 **abstract**, 则这个类必须使用 **abstract** 修饰

```scala
abstract class Person(val name: String) {
    def sayHello: Unit
}

class Student(name: String) extends Person(name) {
    def sayHello: Unit = println("Hello, " + name)
}
```

### 2.3.7 抽象field

- 在父类中，定义了 field，但是没有给初始值，则此field为抽象field, **class 中的 field 必须要赋初值，否则就认为是抽象field，其对应的class必须被定义为 abstract**
- 抽象field，scala 会根据自己的规则，为var或val类型的field生成对应的getter和setter方法，但是父类中是没有该field
- 子类必须覆盖field，以定义自己的具体field，并且覆盖抽象field，不需要使用**override**

```scala
abstract class Person {
    val name: String
}

class Student extends Person{
    val name: String = "bob"
}
```

## 2.4 Trait

### 2.4.1  将 trait 作为接口使用

- 与 JAVA 中的接口类似；
- 在 triat 中定义抽象方法，scala 中没有 **implement** 关键字，一律使用 **extends**，实现抽象方法时不需要使用 **override**；
- scala 不支持多继承 **class**，但是支持多重继承 **trait**， 但是需要使用 **with**:

```scala
trait HelloTrait {
    def sayHello(name: String)
}
trait MakeFriendsTrait {
    def makeFriends(p: Person)
}
class Person(var name: String) extends HelloTrait with MakeFriendsTrait with Cloneable with Serializable {
    def sayHello(otherName: String) = println("Hello, " + otherName + ", I'm " + name)
	def makeFriends(p: Person) = println("Hello, my name is " + name + ", your name is " + p.name)
}
```

### 2.4.2 在 trait 中定义具体方法

- 可以定义一些通用的功能方法

```scala
trait Logger {
    def log(message: String) = println(message)
}
class Person(val name: String) extends Logger {
    def makeFriends(p: Person) {
        println(p.name)
        log(p.name)
    }
}
```

### 2.4.3 在 trait 中定义具体字段

- 继承 trait 的类会自动获得 trait 中定义的 field， 但是与继承 class 不同： 如果继承 class 获取 field， 实际是定义在父类中，子类只是获得了父类中 field 的引用；而继承 trait 获取的field，就被直接添加到了实现类中。

```scala
trait Person {
    val eyeNum: Int = 2
}
class Student(val name: String) extends Person {
    def sayHello = println("Hello, I'm " + name + ", I hava " + eyeNum + " eyes.")
}
```

### 2.4.4  在 trait 中定义抽象字段

- 继承的抽象字段必须得赋值

```scala
trait SayHello {
    val msg: String
    def sayHello(name: String) = println(msg + ", " + name)
}
class Person(val name: String) extends SayHello {
    val msg: String = "hello"
    def makeFriends(p:Person) {
        sayHello(p.name)
        println(name)
    }
}
```

### 2.4.5  作为实例对象混入 trait

- 在创建类的对象时，指定该对象混入某个trait， 这样，就只有这个对象混入该trait的方法，而类的其他对象没有

```scala
trait Logged {
    def log(msg: String) {}
}
trait MyLogger extends Logged {
   override def log(msg: String) {
        println("log:" + msg)
    }   
}
class Person(val name: String) extends Logged {
    def sayHello {
        println("Hi, I'm " + name)
        log("sayHello is invoked!")
    }
}

val p1 = new Person("Li")
p1.sayHello
val p2 = new Person("Zhang") with MyLogger  // 给实例对象动态混入 trait
p2.sayHello
```

### 2.4.6 trait 调用链（责任链模式）

- Scala 中支持让类继承多个trait后，依次调用多个 trait 中的同一个方法， **多个 trait 的同一个方法中，在最后都要执行 super.方法**；
- 类中调用多个trait中都有的这个方法时，会**从右往左**依次执行trait中的方法，形成一个调用链
- 相当于设计模式中的责任链模式

```scala
trait Handler {
    def handler(data: String) {}
}
trait DataValidHandler extends Handler {
    override def handler(data: String) {
        println("check data: " + data)
        super.handler(data) // 关键语句
    }
}
trait SignatureValidHandler extends Handler {
    override def handler(data: String) {
        println("check signature: " + data)
        super.handler(data)
    }
}
class Person(val name: String) extends SignatureValidHandler with DataValidHandler {
    def sayHello = {
        println("Hello, " + name)
        handler(name)
    }
}

val p = new Person("Li")
p.sayHello
```

### 2.4.7  在 trait 中覆盖抽象方法

- 在 trait 中覆盖父 trait 的抽象方法
- 覆盖时，如果使用了 `super.方法` 的代码，则无法通过编译，因为 `super.方法` 会去调用父 trait的抽象方法，此时子 trait 的该方法还是会被认为是抽象的，如果想要通过编译，必须给子 trait 方法加上 `abstract override`。

```scala
trait Logger {
    def log(msg: String)
}
trait MyLogger extends Logger {
    abstract override def log(msg: String) {super.log(msg)}
}
```

### 2.4.8 混合使用 trait 的具体方法和抽象方法（模板设计模式）

- 可以在 trait 中让具体方法依赖于抽象方法，而抽象方法则放到继承trait的类中实现（模板设计模式）

```scala
trait Valid {
    def getName: String
    def valid: Boolean = {
        getName == "leo"
    }
}
class Person(val name: String) extends Valid {
    println(valid)
    def getName = name
}
```



### 2.4.9 trait 的构造机制

- 在 Scala 中，trait 也有构造代码，也就是 trait 中不在任何方法中的代码，会被自动放进 构造代码中；
- 继承了 trait 的类的构造机制如下： 
  - 父类的构造函数执行
  - trait 的构造代码执行，多个trait **从左到右** 依次执行
  - 构造trait时会先构造父 trait，如果多个 trait 继承同一个父 trait，则父 trait 只会构造依次
  - 所有 trait 构造完毕之后，子类的构造函数才会执行

```scala
class Person {println("Person's constructor")}
trait Logger {println("Logger's constructor")}
trait MyLogger extends Logger {println("MyLogger's constructor")}
trait TimeLogger extends Logger {println("TimeLogger's constructor")}
class Student extends Person with MyLogger with TimeLogger {
    println("Student's constructor")
}

Out:
Person's constructor
Logger's constructor
MyLogger's constructor
TimeLogger's constructor
Student's constructor
```

### 2.4.10 trait 字段的初始化

- **trait 与 class 的唯一区别**： trait 没有接收参数的构造函数；

```scala
trait SayHello {
    val msg: String
    println(msg.toString)
}
// 错误
class Person extends SayHello {
    override val msg: String = "init"
}
val p = new Person() // 报错，因为会先执行 println(msg.toString) ，再执行 Person 中的赋值操作，所以会报空指针异常
// 方式一: 提前赋值
class Person 
val p = new {
    val msg: String = "init"
} with Person with SayHello
// 方式二
class Perosn extends {
    val msg:String = "init"
} with SayHello {}
//方式三： 使用 lazy
trait SayHello {
    lazy val msg: String = null
    println(msg.toString)
}
class Person extends SayHello {
    override lazy val msg:String = "init"
}
```

### 2.4.11 trait 继承类

- trait 也可以继承自 class， 此时这个class就会成为所有继承该 trait 的类的父类

```scala
class MyUtil {
    def printMessage(msg: String) = println(msg)
}
trait Logger extends MyUtil {
    def log(msg: String) = printMessage("log: " + msg)
}
class Person(val name: String) extends Logger {
    def sayHello {
        log("Hi, I'm " + name)
        printMessage("Hi, I'm " + name)
    }
}
```


# 3. Scala 函数式编程

# 4. Scala 高级特性

# 5. Scala Actor 并发编程



