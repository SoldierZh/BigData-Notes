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

- **while** ： 使用与 Java 相同 
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

- **trait 与 class 的唯一区别**： trait 没有接收参数的构造函数，有时需要提前初始化 trait 字段

```scala
trait SayHello {
    val msg: String
    println(msg.toString)
}
// 错误
class Person extends SayHello {
    override val msg: String = "init"
}
val p = new Person() // 报错，因为会先执行 println(msg.toString)【此时msg还没有赋值】，再执行 Person 中的赋值操作，所以会报空指针异常
// 方式一: 提前初始化
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

> Java 是面向对象编程语言，方法不能脱离类和对象独立存在；Scala 是既可以面向对象，又可以面向过程的语言。
>
> **注**： **方法** 是对象或者类中的， **函数** 是独立存在的。



## 3.1  将函数赋值给变量

- Scala 语法规定，将函数赋值给变量时，必须在函数后面加上空格和下划线：

```scala
def sayHello(name: String) = println("Hello, " + name)
val sayHelloFunc = sayHello _
```



## 3.2 匿名函数

- 可以直接定义函数之后，将函数赋值给某个变量，或者其他函数的参数中;
- (参数名： 参数类型)  => 函数体

```scala
val sayHelloFunc = (name: String) => println("Hello, " + name)
```



## 3.3 高阶函数

- Scala 可以将函数作为参数传入其他函数，接收其他函数的被称为高阶函数

```scala
// 参数类型高阶函数
val sayHelloFunc = (name: String) => println("Hello, " + name)
def greeting(func: (String) => Unit, name: String) {func(name)}
greeting(sayHelloFunc, "hello")
```

```scala
Array(1,2,3,4,5).map((num: Int) => num * num)
```

```scala
// 返回值类型高阶函数
def getGreetingFunc(msg: String) = (name: String) => println(msg + ", " + name)
val greetingFunc = getGreetingFunc("Hello")
greetingFunc("leo")
```



## 3.4 高阶函数的类型推断

- 高阶函数可以自动推断出参数类型，不需要写明类型
- 对于只有一个参数的函数，可以省去小括号
- 如果仅有一个参数在右侧的函数体内只使用一次，可以将接受参数省略，将参数用_来替代

```scala
def greeting(func: (String) => Unit, name: String) {func(name)}
greeting((name: String) => println("Hello, " +name), "Li")
greeting((name) => println("Hello, " + name), "Li") // 不需要写明匿名函数的参数类型
greeting(name => println("Hello, " + name), "Li") // 一个参数，可以去除匿名函数的参数括号
def triple(func: (Int) => Int) = {func(3)}
triple(num => 5 * num)
triple(5 * _) // 匿名函数仅有一个参数，并且在函数体内只使用一次，可以将参数省略('num =>'省略)，将参数用_替代(使用 _ 替代 num)
```



## 3.5 Scala的常用高阶函数

### 3.5.1 map

- 对传入的每个元素都进行映射，返回一个处理后的元素

```scala
//以下四种效果一样
Array(1,2,3,4,5).map(2 * _)
Array(1,2,3,4,5).map(num => 2 * num)
Array(1,2,3,4,5).map((num) => num * 2)
Array(1,2,3,4,5).map((num: Int) => num * 2)
```

### 3.5.2 foreach

- 对传入的每个元素都进行处理，但是没有返回值

```scala
(1 to 9).map("*" * _).foreach(println _)
```

### 3.5.3 filter

- 对传入的每个元素都进行判断，如果为 ture, 则会保留

```scala
(1 to 20).filter(_ % 2 == 0)
```

### 3.5.4 reduceLeft

- 从左侧元素开始，进行reduce操作，即先对元素1 和元素 2进行处理，然后将结果与元素3处理，再将结果与元素4处理，依次类推，即为reduce

```scala
(1 to 5).reduceLeft(_ * _) // 相当于： 1*2*3*4*5
```

### 3.5.5 sortWith

- 对元素进行俩俩相比，进行排序

```scala
Array(3,2,5,4,10,1).sortWith(_ < _)
```



## 3.6 闭包

> 函数在变量不处于其有效作用域时，还能够对变量进行访问，即为闭包

```scala
def getGreetingFunc(msg: String) = (name: String) => println(msg + ", " + name) 
val greetingFuncHello = getGreetingFunc("hello")
val greetingFuncHi = getGreetingFunc("Hi")
```

> 两次调用 getGreetingFunc 函数，传入不同的 msg， 并创建不同的函数返回，然而，msg 只是 getGreetingFunc 函数的局部变量，却在 getGreetingFunc 执行完之后，还可以继续存在于创建的函数中，这种变量超出了其作用域，还可以继续使用，即为闭包。

- Scala 通过为每个函数创建对象来实现闭包，实际上对于 getGreetingFunc 函数创建的函数，msg是作为函数对象的变量存在的，因此每个函数才可以拥有不同的 msg。



## 3.7 SAM 转换

> 将Java方法转换为Scala的匿名函数

- 在Java中，不支持直接将函数传入一个方法作为参数，唯一的办法就是定义一个实现了某个接口的类的实例对象，该对象只有一个方法；而这些接口都只有单个抽象方法，也就是 single abstract method （SAM）
- 由于 Scala 是可以调用Java的代码的，因此当调用Java的某个方法时，可能就不得不创建SAM传递给方法，非常麻烦；但是Scala 又是支持直接传递函数的。此时就可以使用Scala提供的，在调用Java方法时，使用的功能，SAM转换，即将SAM转为Scala函数
- 要使用SAM转换，需要使用Scala提供的特性，隐式转换

```scala
import javax.swing._
import java.awt.event._
val button = new JButton("Click")
button.addActionListener(new ActionLister {
    override def actionPerformed(event: ActionEvent) {
        println("Click Me!!!")
    }
})
// SAM
implicit def getActionListener(actionProcessFunc: (ActionEvent) => Unit) = new ActionListener{
    override def actionPerformed(event: ActionEvent) {
        actionProcessFunc(event)
    }
}
button.addActionListener((event: ActionEvent) => println("Click Me!!!"))
```

​	

## 3.8 Currying 函数

- 将原来接收的两个参数的一个函数，转换为两个函数，第一个函数接收原来的第一个参数，然后返回接收原先第二参数的第二个函数
- 在函数调用过程中，就变成了两个函数连续两个函数调用的形式

```scala
def sum(a: Int, b: Int) = a + b
sum(1, 1)
// Currying 函数
def sum2(a: Int) = (b: Int) => a + b
sum2(1)(1)
// Currying 函数简写
def sum3(a: Int)(b: Int) = a + b
```



## 3.9 return

- 在 Scala 中， return 用于在匿名函数中返回值给包含匿名函数的带名函数，并作为带名函数的返回值 。
- 使用 return 的匿名函数，是必须给出返回类型的，否则无法通过编译

```scala
def greeting(name: String) = {
    def sayHello(name: String): String = {
        return "Hello, " + name
    }
    sayHello(name)
}
```

# 4. Scala 高级特性

## 4.1 函数式编程之集合操作

### 4.1.1 Scala 的集合体系结构

- Scala 中的集合体系主要包括：**Iterable**、**Seq**、**Set**、**Map**。其中 **Iterable** 是所有集合 trait 的根 trait。这个结构与Java的集合体系非常相似。
- Scala 中的集合分成可变和不可变两类集合，分别对应 `scala.collection.mutable` 和 `scala.collection.immutable` 两个包
- **Seq** 下包含了 **Range**、**ArrayBuffer**、**List** 等 trait。其中 Range 就代表了一个序列，通常可以使用 `1 to 10` 语句产生一个 Range。 ArrayBuffer类似于Java的ArrayList。

### 4.1.2 List

- List 代表一个不可变的数组
- List 的 head 和 tail ， head 代表List的第一个元素， tail 代表第一个元素之后的所有元素
- List 的特殊操作符 `::` 可以将 head与tail合并成一个list， `0::list` 

```scala
def decorator(list: List[Int], prefix: String) {
    if(list != Nil) {
        println(prefix + list.head)
        decorator(list.tail, prefix)
    }    
}
```

### 4.1.3 LinkedList

- LinkedList 代表一个可变的列表，使用 `elem` 可以引用其头部(head)，使用 `next` 可以引用其尾部( tail )

```scala
// 使用while循环将 list中的每个元素乘以 2
var list = scala.collection.mutable.LinkedList(1,2,3,4,5)
var currentList = list
while(currentList != Nil) {
	currentList.elem = currentList.elem * 2
    currentList = currentList.next
}
// 使用while循环将 list 中每隔一个元素乘以2
var list = scala.collection.mutable.LinkedList(1,2,3,4,5)
var currentList = list
var first = true
while(currentList != Nil && currentList.next != Nil) {
    if(first) {
        currentList.elem = currentList.elem * 2
        first = false
    }
    currentList = currentList.next.next
    currentList.elem = currentList.elem * 2
}
```

### 4.1.4 Set

- Set 代表一个没有重复元素的集合
- HashSet 不能保证插入的顺序
- LinkedHashSet 可以保证插入顺序
- SortedSet 会自动根据 key 来进行排序

### 4.1.5 集合的函数式编程

-  map、flatMap、reduce、reduceLeft、foreach、zip

### 4.1.6 函数式编程综合案例：统计多个文本内的单词总数

```scala
val line1 = "hello world"
val line2 = "hello you"
val lines = List(line1, line2)
lines.flatMap(_.split(" ")).map((_, 1)).map(_._2).reduceLeft(_ + _)
```

## 4.2  模式匹配

- **match case** 类似Java中的 **switch case** 语法
- Scala 模式匹配除了可以对值进行匹配， 还可以对类型、Array、List 等进行匹配

### 4.2.1 基础语法

>  `match{case 值 => 语法}` ，如果值是下划线，则代表了不满足以上所有情况下的默认情况如何处理，只要一个 case 条件满足并处理了，就不会继续向下匹配其他case分枝。

- 基本语法

```scala
def judgeGrade(grade: String) {
    grade match {
        case "A" => println("Excellent")
        case "B" => println("Good")
        case "C" => println("Just so so")
        case _ => println("you need to work harder")
    }
}
```

- 带有if守卫的模式匹配，相当于对匹配条件进双重过滤，进一步细化

```scala
def judgeGrade(name: String, grade: String) {
    grade match {
        case "A" => println("Excellent")
        case "B" => println("Good")
        case "C" => println("Just so so")
        case _ if name == "Li" => println(name + ", come on")
        case _ => println("you need to work harder")
    }
}
```

- 在模式匹配中进行变量赋值：将模式匹配的默认情况，下划线，替换成一个变量名，此时模式匹配语法就会将要匹配的值赋值给这个变量，从而可以在后面的处理语句中使用要匹配的值

```scala
def judgeGrade(name: String, grade: String) {
    grade match {
        case "A" => println("Excellent")
        case "B" => println("Good")
        case "C" => println("Just so so")
        case _grade if name == "Li" => println(name + ", come on")
        case _grade => println("you need to work harder, you just got " + _grade)
    }
}
```

### 4.2.2 对类型进行模式匹配

- 可以直接匹配类型，而不是值
- `case 变量:类型 => 代码` 

```scala
import java.io._
def processException(e: Exception) {
    e match {
        case e1: IllegalArgumentExcetion => println("argument is illegal")
        case e2: FileNotFoundException => println("cannot find file!!!")
        case _: Exception => println("other exception")
    }
}
```

### 4.2.3 对Array和List进行模式匹配

- 对Array进行模式匹配，匹配带有指定元素的数组、带有指定个数元素的数组、以某元素开头的数组；
- 对List进行模式匹配，与Array类似，但是需要使用List特有的 :: 操作符

```scala
def greeting(arr: Array[String]) {
    arr match {
        case Array("Leo") => println("Hello, Leo!") // 带有指定元素的数组
        case Array(p1, p2, p3) => println("people: " + p1 + ", " + p2 + ", " + p3) // 带有指定个数元素的数组
        case Array("Wang", _*) => println("Hello Wang!") //  以某元素开头的数组 
    }
}
def greeting(list: List[String]) {
    list match {
        case "Leo" :: Nil => println("Hello, Leo!")
        case p1 :: p2 :: p3 :: Nil => println("people: " + p1 + ", " + p2 + ", " + p3)
        case "Wang" :: tail => println("Hello Wang!")
        case _ => println("Who are you?") 
    }
}
```

### 4.2.4 case class 与模式匹配

- Scala 提供了一种特殊的类，用 `case class` 进行声明， 有点类似Java中的 JavaBean，只定义 field，并由 Scala 编译时自动提供getter和setter方法，但是没有method。
- case class 的主构造函数接收的参数通常不需要使用var或者val修饰，Scala 会自动使用 val 修饰，如果指定var修饰，还是会按照 var
- Scala自动为 case class 定义了伴生对象，也就是 object， 并且定义了 apply() 方法

```scala
class Person
case class Teacher(name: String, subject: String) extends Person
case class Student(name: String, classroom: String) extends Person
//
def judgeIdentify(p: Person) {
    p match {
        case Teacher(name, subject) => println("Teacher, name = " + name + ", subject = " + subject) 
        case Student(name, classroom) => println("Student, name = " + name + ", classroom = " + classroom)
        case _ => println("Illegal access")
    }
}
```

### 4.2.5 Option 与模式匹配

- Scala 的一种特殊类型，**Option**， 有两种值：Some（有值）、None（没有值）
-  Option 通常会用于模式匹配中，用于判断某个变量是否有值

```scala
val grades = Map("Leo" -> "A", "Jack" -> "B")
def getGrade(name: String) {
    val grade = grades.get(name)
    grade match {
        case Some(grade) => println("your grade is " + grade)
        case None => println("not found")
    }
}
```

## 4.3 泛型

> 与Java的泛型概念一样

### 4.3.1 泛型类

```scala
class Student[T](val localId: T) {
    def getSchoolId(hukouId: T) = "S-" + hukouId + "-" + localId
}
 val student0 = new Student(123)
 val student1 = new Student[Int](123)
 val student2 = new Student[String]("123")
```

### 4.3.2 泛型函数

```scala
def getCard[T](content: T) = {
    if(content.isInstanceOf[Int]) "card:001, " + content
    else if(content.isInstanceOf[String]) "card: this is your card, " + content
    else "card: " + content
}
getCard[String]("hello")
getCard("hello")
```

### 4.3.3 上边界 Bounds

- 类似 Java 的 `T extends Person`
- `<:`

```scala
class Person(val name: String) {
    def sayHello = println("Hello, I'm " + name)
    def makeFriends(p: Person) {
        sayHello
        p.sayHello
    }
}
class Student(name: String) extends Person(name)
class Party[T <: Person](p1: T, p2: T) {
    def play = p1.makeFriends(p2)
}
// test
val person = new Person("person")
val student = new Student("student")
val party = new Party(person, student)
party.play
```

### 4.3.4 下边界 Bounds

- 类似 Java 的 `T super Father`
- `>:`

```scala
class Father(val name: String)
class Child(name: String) extends Father(name) 
def getIDCard[R >: Child](person: R) {
    if(person.getClass == classOf[Child]) println("tell us your father's name")
    else if(person.getClass == classOf[Father]) println("sign your name")
    else println("sorry")
}
```

### 4.3.5 View Bounds

- 上下边界Bounds的加强版，支持可以对类型进行隐式转换，将指定的类型进行**隐式转换**后，再判断是否在边界指定类型范围内
- `<%`

```scala
import scala.language.implicitConversions
class Person(val name: String) {
    def sayHello = println("Hello, I'm " + name)
    def makeFriends(p: Person) {
        sayHello
        p.sayHello
    }
}
class Student(name: String) extends Person(name)
class Dog(val name: String) {
    def sayHello = println("Wang! I'm" + name)
}
// 隐式类型转换，将Dog转为Person
implicit def dog2person(dog: Object): Person = {
    if(dog.isInstanceOf[Dog]) {
        val _dog = dog.asInstanceOf[Dog]
        new Person(_dog.name)
    } else {
        Nil
    }  
} 
class Party[T <% Person](p1: T, p2: T) {
    def play = p1.makeFriends(p2)
}
val student = new Student("student")
val dog = new Dog("dog")
```

### 4.3.6 Context Bounds

- 一种特殊的Bounds，它会根据泛型类型的声明，比如 **T :类型** 要求必须存在一个类型为**类型[T]**的隐式值。
- `T: Ordering`

```scala
class Calculator[T: Ordering] (val number1: T, val number2: T) {
    def max(implicit order: Ordering[T]) = {
        if(order.compare(number1, number2)>0)
        	number1
        else
        	number2
    }
}
```

### 4.3.7 Manifest Context Bounds

- 如果要实例化一个**泛型数组**， 就必须使用 Manifest Context Bounds。如果数组元素类型为 T， 需要为类或者函数定义 `[T: Manifest]` 泛型类型，这样才能实例化  `Array[T]` 这种泛型数组
- `T: Manifest` 

```scala
class Meat(val name: String) 
class Vegetable(val name: String)
def packageFood[T: Manifest](food: T*) = {
    val foodPackage = new Array[T](food.length)
    for(i <- 0 until food.length)
    	foodPackage(i) = food(i)
    foodPackage
}
packageFood(new Meat("m1"), new Meat("m2"), new Meat("m3"))
 packageFood(new Vegetable("p1"), new Vegetable("p2"), new Vegetable("p3"))
```

### 4.3.8 协变和逆变

- Java 中 ArrayList<String> 不是 ArrayList<Object> 的子类， Scala 的协变和逆变可以解决这个问题
- `+T`  `-T`

```scala
class Master
class Professional extends Master
//协变： 向子类兼容
class Card[+T](val name: String) 
def enterMeeting(card: Card[Master]) {
    println("welcome " + card.name)
}
enterMeeting(new Card[Master]("master"))
enterMeeting(new Card[Professional]("professional"))
//逆变：向父类兼容
class Card[-T](val name: String)
def enterMeeting(card: Card[Professional]) {
    println("Welcome " + card.name)
}
enterMeeting(new Card[Master]("master"))
enterMeeting(new Card[Professional]("professional"))
```

### 4.3.9 Existential Type

```scala
Array[T] forSome{type T}
Array[_]
```

## 4.4 隐式转换与隐式参数

- 允许手动指定，将某种类型的对象转换成其他类型的对象。

- Scala 隐式转换，其核心就是定义**隐式转换函数**， 即 **implicit conversion function**，只要在编写的程序内引入，就会被Scala自动使用。Scala会根据隐式转换函数的签名，在程序中使用到隐式转换函数接收的参数类型定义的对象时，会自动将其传入隐式转换函数，转换为另外类型的对象并返回。

- 隐式转换函数命名无所谓，因为通常不会由用户手动调用，而是由Scala进行调用，但是如果使用隐式转换，则需要对隐式转换函数进行**导入**。通常建议将隐式转换函数命名为 ”one2one“的形式。

### 4.4.1 隐式转换

> 使用 **implicit** 关键字开头的函数，最好定义函数返回类型

```scala
import scala.language.implicitConversions 
class SpecialPerson(val name: String)
class Student(val name: String)
class Older(val name: String)
// 隐式转换函数
implicit def object2SpecialPerson(obj: Object): SpecialPerson = {
    if(obj.getClass == classOf[Student]) {
        val stu = obj.asInstanceOf[Student]
        new SpecialPerson(stu.name)
    } else if(obj.getClass == classOf[Older]) {
        val older = obj.asInstanceOf[Older]
        new SpecialPerson(older.name)
    } else {
        Nil
    }
}
var ticketNumber = 0
def buySpecialTicket(p: SpecialPerson) = {
    ticketNumber += 1
    "T-" + ticketNumber
}
// test
val student = new Student("student")
buySpecialTicket(student)  // 隐式转换
```

### 4.4.2 隐式转换加强现有类型

```scala
import scala.language.implicitConversions
class Man(val name: String)
class Superman(val name: String) {
    def emitLaser = println("emit a laser!")
}
implicit def man2superman(man: Man): Superman = new Superman(man.name)
// test
val zhang = new Man("zhang")
zhang.emitLaser  // 调用隐式转换函数
```

### 4.4.3 导入隐式转换函数

- Scala 默认会使用两种隐式转换， 一种是源类型，或者目标类型的伴生对象内的隐式转换函数；一种是当前程序作用域内的可以用唯一标识符表示的隐式转换函数。
- 如果隐式转换函数不在上述两种情况下，那么则需要手动使用 `import` 语法引入某个包下的隐式转换函数。

### 4.4.4 隐式转换的发生时机

1. 调用某个函数，但是给函数传入的参数的类型，与函数定义的接收参数类型不匹配；
2. 使用某个类型的对象，调用某个方法并不存在于该类型时
3. 使用某个类型的对象，调用某个方法，虽然该类型有这个方法，但是给方法传入的参数类型，与方法定义的接收参数的类型不匹配（与第一种类似）

### 4.4.5 隐式参数

- 在函数或者方法中，定义一个用 `implicit` 修饰的参数，此时 Scala 会尝试找到一个指定类型的，用 `implicit` 修饰的对象，即隐式值，并注入参数
- Scala 会在两个范围内查找： 一种是当前作用域内可见的val或者var定义的隐式变量； 一种是隐式参数类型的伴生对象内的隐式值

```scala
class SignPen {
    def write(content: String) = println(content)
}
implicit val signPen = new SignPen
def signForExam(name: String)(implicit signPen: SignPen) {
    signPen.write(name + " come to exam in time.")
}
```

# 5.  Actor 并发编程入门

- Actor 类似Java中**多线程编程**，但与之不同的是： Scala的Actor尽可能地避免锁和共享状态，从而避免多线程并发时出现资源争用的情况，从而提升多线程编程的性能，同时还可以避免死锁等一系列传统多线程编程的问题。
- Spark 中使用的分布式多线程框架，是 Akka。 Akka 也实现了类似 Scala Actor 的模型，其核心概念同样也是Actor。
- **注意** ： 从Scala的2.11.0版本开始，Scala的Actors库已经过时了。早在Scala2.10.0的时候，默认的actor库即是Akka。

## 5.1 Actor 的 Hello World

- Scala 中的 **Actor trait** 类似 Java 中的 **Runnable**
-  需继承**Actor** triat 并重写 **act**方法
-  使用 `!` 符号 向actor 发送消息
-  在 actor内部使用 `receive` 和模式匹配接收消息

```scala
import scala.actors.Actor
class HelloActor extends Actor {
    def act() {
        while(true) {
            receive{
                case name: String => println("Hello, " + name)
            }
        }
    }
}
val helloActor = new HelloActor
helloActor.start()
// 发送消息
helloActor ! "wang"
```

## 5.2 收发 case class 类型消息

- 相当于给 Actor 发送一个 JavaBean

```scala
case class Login(username: String, password: String)
case class Register(username: String, password: String)
class UserManagerActor extends Actor {
    def act() {
        while(true) {
            receive {
                case Login(username, password) => println("login, username: " + username + ", password: " + password)
                case Register(username, password) => println("resiger, username: " + username + ", password: " + password)
            }
        }
    }
}
val userManagerActor = new UserManagerActor
userManagerActor.start()
userManagerActor ! Register("leo", "1234")
userManagerActor ! Login("leo", "1234")
```

## 5.3 Actor 之间互相收发消息

- actor1 给 actor2 发送消息时，包含自己的引用，actor2 可以根据 actor 引用直接回复消息

```scala
case class Message(content: String, sender: Actor)
class LeoTelephoneActor extends Actor {
    def act() {
        while(true) {
            receive {
                case Message(content, sender) => {println("leo telephone: " + content); sender ! "please call me after 10 mins."}
            }
        }
    }
}
class JackTelephoneActor(val leoTelephoneActor: Actor) extends Actor {
    def act() {
    	leoTelephoneActor ! Message("Hello, Leo, I'm Jack.", this)
    	receive {
            case response: String => println("jack telephone: " + response)
    	}
    }
}
```

## 5.4 同步消息和 Future

- 默认情况下，消息都是异步的，但是希望同步，可以使用 `!?` 发送消息，例如： `val reply = actor !? message`，阻塞直到获取返回值
- 如果要异步发送一个消息，在后续要获得消息的返回值，可以使用 Future，即 `!!` , 例如发送消息时： `val future = actor !! message`， 获取返回值： `val reply = future()`


