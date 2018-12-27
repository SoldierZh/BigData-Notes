# 1. Scala 基本语法

## 1.1 环境搭建

- [Scala官网](https://www.scala-lang.org/download/)  下载 [scala-2.12.8.msi](https://downloads.lightbend.com/scala/2.12.8/scala-2.12.8.msi) 

- 安装 Scala 并配置 环境变量 ， 前提需要安装 JDK 1.8

- 进入CMD，运行 scala 命令，进入scala命令行

## 1.2 Scala 解释器

- **REPL**： Read -> Evaluation -> Print ->Loop；
- Scala 底层是 JVM， 所有Scala源码需要编译成 class文件，然后才可以执行；

## 1.3 声明变量

- val ：声明常量， `val a = 1`
- var ：声明变量， `var b = 2` 
- 指定类型： `var b:String = "hello"`
- 声明多个变量： `var name1,name2:String = null`

## 1.4 数据类型与操作符

- 基本数据类型：Byte、Char、Short、Int、Long、Float、Double、Boolean；
- 增强版基本数据类型：StringOps、RichInt、RichDouble、RichChar，提供了更多功能函数， 基本数据类型在必要时会隐式转换为其对应的增强版类型；
- 基本操作符：与Java的不同点在于，没有++、-- 运算符； 1+1与 1.+(1) 效果相同，相当于调用函数。

## 1.5 函数调用与apply函数

- 函数调用方式与Java相似，不同点在于，Scala中如果函数没有参数，则调用时可以不写括号；
- **apply** 函数：自定义类中可以重写apply函数，“类名()” 的形式，其实就是 “类名.apply()” 的缩写，用来创建一个对象，而不使用 “new 类名()” 的方式创建对象 。

## 1.6 条件控制 if else

-  带返回值的语句： `val type = if(age > 18) 1 else 0`， Scala会自动推断返回类型，取 if 和 else 的公共父类型，如 `var type = if(age > 18) 1 else "no"` ， type的类型为 Any，如果没有 else语句，则相当于 `else ()` ；
-  不带返回值的语句： `if(age > 18) type = 1 else type = 0` ;
-  if 子句包含多行：需要使用 `{ }` ，或者进入 `:paste` 编辑之后，再 `ctrl+D`的方式执行。

## 1.7 输入输出

- **readLine()** ： `val name = readLine("please show your name :\n")`, `var name = readLine()`,  或者 `var name = io.StdIn.readLine()`;
- **readInt()** : `val age = readInt()`;
- **print()**： `print("Hello World")`，相当于 Java 的 `System.out.print()`；
- **println()**：带换行的输出，相当于 Java 的 `System.out.println()`；
- **printf()**：格式化输出， `printf("your age is %d" , age)`，相当于 Java 的 `System.out.printf()`；

## 1.8 循环语句 

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

## 1. 9 函数入门

- 定义函数
```scala
def sayHello(name: String, age: Int) = {
    if(age >= 18) {
        printf("Hi, %s, you are a big boy", age)
    } else {
        printf("Hi, %s, you are a child", age)
    }
}
```



# 2. Scala 面向对象编程

# 3. Scala 函数式编程

# 4. Scala 高级特性

# 5. Scala Actor 并发编程



