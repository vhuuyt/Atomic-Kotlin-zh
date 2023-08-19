# 附录 B：Java 互操作性

> 本附录描述了在 Kotlin 和 Java 之间进行接口调用的问题和技巧。

Kotlin 的一个重要设计目标是为 Java 程序员创建无缝的体验。如果你想逐步迁移到 Kotlin，你可以轻松地在现有的 Java 项目中添加一些 Kotlin 代码。这样，你可以在现有的 Java 代码基础上编写新的 Kotlin 代码，享受 Kotlin 语言的特性，而无需在不合适的情况下强制重写 Java 代码。

不仅可以轻松地从 Kotlin 调用 Java 代码，还可以直接在 Java 程序中调用 Kotlin 代码。

### 从 Kotlin 调用 Java

要从 Kotlin 中使用 Java 类，只需像在 Java 中一样，导入它，创建一个实例，并调用函数。下面的示例演示了如何使用 `java.util.Random()`：

```kotlin
// interoperability/Random.kt
import atomictest.eq
import java.util.Random

fun main() {
  val rand = Random(47)
  rand.nextInt(100) eq 58
}
```

与在 Kotlin 中创建任何实例一样，你不需要使用 Java 中的 `new` 关键字。Java 库中的类可以像本地的 Kotlin 类一样工作。

Java 类中的 JavaBean 风格的 getter 和 setter 方法在 Kotlin 中变成属性：

```java
// interoperability/Chameleon.java
package interoperability;
import java.io.Serializable;

public
class Chameleon implements Serializable {
  private int size;
  private String color;
  public int getSize() {
    return size;
  }
  public void setSize(int newSize) {
    size = newSize;
  }
  public String getColor() {
    return color;
  }
  public void setColor(String newColor) {
    color = newColor;
  }
}
```

在使用 Java 时，包名必须与目录名相同（包括大小写）。Java 包名通常只包含小写字母。为了符合这个约定，本附录中的示例子目录名`interoperability`仅使用小写字母。

导入的 `Chameleon` 类的使用方式类似于具有属性的 Kotlin 类：

```kotlin
// interoperability/UseBeanClass.kt
import interoperability.Chameleon
import atomictest.eq

fun main() {
  val chameleon = Chameleon()
  chameleon.size = 1
  chameleon.size eq 1
  chameleon.color = "green"
  chameleon.color eq "green"
  chameleon.color = "turquoise"
  chameleon.color eq "turquoise"
}
```

当你使用一个现有的缺少所需成员函数的 Java 库时，扩展函数尤其有用。例如，我们可以给 `Chameleon` 添加一个 `adjustToTemperature()` 操作：

```kotlin
// interoperability/ExtensionsToJavaClass.kt
package interop
import interoperability.Chameleon
import atomictest.eq

fun Chameleon.adjustToTemperature(
  isHot: Boolean
) {
  color = if (isHot) "grey" else "black"
}

fun main() {
  val chameleon = Chameleon()
  chameleon.size = 2
  chameleon.size eq 2
  chameleon.adjustToTemperature(isHot = true)
  chameleon.color eq "grey"
}
```

Kotlin 标准库中包含了许多针对 Java 标准库中的类（如 `List` 和 `String`）的扩展函数。

### 从 Java 调用 Kotlin

Kotlin 生成的库可以在 Java 中使用。对于 Java 程序员来说，一个 Kotlin 库看起来就像一个 Java 库。

由于在 Java 中一切都是类，让我们从一个包含属性和函数的 Kotlin 类开始：

```kotlin
// interoperability/KotlinClass.kt
package interop

class Basic {
  var property1 = 1
  fun value() = property1 * 10
}
```

如果你将这个类导入到 Java 中，它看起来就像一个普通的 Java 类：

```java
// interoperability/UsingKotlinClass.java
package interoperability;
import interop.Basic;
import static atomictest.AtomicTestKt.eq;

public class UsingKotlinClass {
  public static void main(String[] args) {
    Basic b = new Basic();
    eq(b.getProperty1(), 1);
    b.setProperty1(12);
    eq(b.value(), 120);
  }
}
```

`property1` 变成了一个包含 JavaBean 风格的 getter 和 setter 的 `private` 字段。`value()` 成员函数变成了一个同名的 Java 方法。

我们还导入了 `AtomicTest`，在 Java 中需要额外的步骤：我们必须使用 `static` 关键字导入它，并指定包名。`eq()` 只能作为普通函数调用，因为 Java 不支持中缀表示法。

如果一个 Kotlin 类与 Java 代码位于同一个包中，你就不需要导入它：

```kotlin
// interoperability/KotlinDataClass.kt
package interoperability

data class Staff(
  var name: String,
  var role: String
)
```

`data` 类会生成额外的成员函数，比如 `equals()`、`hashCode()` 和 `toString()`，在 Java 中都能无缝使用。在 `main()` 的末尾，我们通过将一个 `Data` 对象放入 `HashMap`，然后检索它，来验证 `equals()` 和 `hashCode()` 的实现：

```java
// interoperability/UseDataClass.java
package interoperability;
import java.util.HashMap;
import static atomictest.AtomicTestKt.eq;

public class UseDataClass {
  public static void main(String[] args) {
    Staff e = new Staff(
      "Fluffy", "Office Manager");
    eq(e.getRole(), "Office Manager");
    e.setName("Uranus");
    e.setRole("Assistant");
    eq(e,
      "Staff(name=Uranus, role=Assistant)");

    // Call copy() from the data class:
    Staff cf = e.copy("Cornfed", "Sidekick");
    eq(cf,
      "Staff(name=Cornfed, role=Sidekick)");

    HashMap<Staff, String> hm =
      new HashMap<>();
    // Employees work as hash keys:
    hm.put(e, "Cheerful");
    eq(hm.get(e), "Cheerful");
  }
}
```

如果你使用命令行来运行包含 Kotlin 代码的 Java 代码，你必须将 `kotlin-runtime.jar` 作为依赖项包含进来，否则你将会遇到运行时异常，指出找不到某些库实用类。IntelliJ IDEA 会自动包含 `kotlin-runtime.jar`

。

Kotlin 顶层函数会映射到一个以 Kotlin 文件命名的 Java 类中的 `static` 方法：

```kotlin
// interoperability/TopLevelFunction.kt
package interop

fun hi() = "Hello!"
```

要导入该函数，需要指定 Kotlin 生成的类名。在调用该 `static` 方法时，也必须使用这个类名：

```java
// interoperability/CallTopLevelFunction.java
package interoperability;
import interop.TopLevelFunctionKt;
import static atomictest.AtomicTestKt.eq;

public class CallTopLevelFunction {
  public static void main(String[] args) {
    eq(TopLevelFunctionKt.hi(), "Hello!");
  }
}
```

如果你不想用包名限定 `hi()`，可以像我们在 `AtomicTest` 中所做的那样使用 `import static`：

```java
// interoperability/CallTopLevelFunction2.java
package interoperability;
import static interop.TopLevelFunctionKt.hi;
import static atomictest.AtomicTestKt.eq;

public class CallTopLevelFunction2 {
  public static void main(String[] args) {
    eq(hi(), "Hello!");
  }
}
```

如果你不喜欢 Kotlin 生成的类名，可以使用 `@JvmName` 注解来改变它：

```kotlin
// interoperability/ChangeName.kt
@file:JvmName("Utils")
package interop

fun salad() = "Lettuce!"
```

现在，我们使用 `Utils` 来代替 `ChangeNameKt`：

```java
// interoperability/MakeSalad.java
package interoperability;
import interop.Utils;
import static atomictest.AtomicTestKt.eq;

public class MakeSalad {
  public static void main(String[] args) {
    eq(Utils.salad(), "Lettuce!");
  }
}
```

你可以在[文档](https://kotlinlang.org/docs/reference/java-to-kotlin-interop.html)中找到更多细节。

### 适应 Java 到 Kotlin

Kotlin 的一个设计目标是将现有的 Java 类型适应到你的需求中。这种能力不仅限于库设计者，相同的逻辑也可以应用于任何外部代码库。

在 [Recursion](./se04-ch11.md) 中，我们创建了 `Fibonacci.kt` 来高效地生成斐波那契数。该实现受到它返回的 `Long` 大小的限制。如果你想返回更大的值，Java 标准库中包含了 `BigInteger` 类。只需几行代码就可以将 `BigInteger` 转换成感觉像是本机 Kotlin 类的东西：

```kotlin
// interoperability/BigInt.kt
package biginteger
import java.math.BigInteger

fun Int.toBigInteger(): BigInteger =
  BigInteger.valueOf(toLong())

fun String.toBigInteger(): BigInteger =
  BigInteger(this)

operator fun BigInteger.plus(
  other: BigInteger
): BigInteger = add(other)
```

`toBigInteger()` 扩展函数通过调用 `BigInteger` 构造函数并将接收者字符串作为参数来将任何 `Int` 或 `String` 转换为 `BigInteger`。

通过重载 `BigInteger.plus()` 操作符，你可以写成 `number + other`。这

使得与 `BigInteger` 的处理相比 Java 中笨拙的 `number.plus(other)` 更加愉快。

使用 `BigInteger`，`Recursion/Fibonacci.kt` 轻松转换为生成更大的结果：

```kotlin
// interoperability/BigFibonacci.kt
package interop
import atomictest.eq
import java.math.BigInteger
import java.math.BigInteger.ONE
import java.math.BigInteger.ZERO

fun fibonacci(n: Int): BigInteger {
  tailrec fun fibonacci(
    n: Int,
    current: BigInteger,
    next: BigInteger
  ): BigInteger {
    if (n == 0) return current
    return fibonacci(
      n - 1, next, current + next)   // [1]
  }
  return fibonacci(n, ZERO, ONE)
}

fun main() {
  (0..7).map { fibonacci(it) } eq
  "[0, 1, 1, 2, 3, 5, 8, 13]"
  fibonacci(22) eq 17711.toBigInteger()
  fibonacci(150) eq
    "9969216677189303386214405760200"
      .toBigInteger()
}
```

所有的 `Long` 都被替换为 `BigInteger`。在 `main()` 中，你可以看到使用不同的 `toBigInteger()` 扩展函数将 `Int` 和 `String` 转换为 `BigInteger`。在第 **[1]** 行中，我们使用 `plus` 操作符来计算 `current + next` 的和；这与使用 `Long` 的原始版本完全相同。

`fibonacci(150)` 在 `Recursion/Fibonacci.kt` 版本中会溢出，但在转换为 `BigInteger` 后可以正常工作。

### Java Checked Exceptions & Kotlin

Java引入了被称为“checked exceptions”的特性，这使得在调用函数时必须捕获每个指定的异常。在Java中，如果你不在try块中处理这些异常，编译时会产生错误。这在某种程度上增加了代码的安全性，但也增加了编码的复杂性。

下面是一个在Java中打开、读取和关闭文件的示例，演示了checked exceptions的使用：

```java
// interoperability/JavaChecked.java
package interoperability;
import java.io.*;
import java.nio.file.*;
import static atomictest.AtomicTestKt.eq;

public class JavaChecked {
  // 根据在Gradle调用的目录构建当前源文件的路径:
  static Path thisFile = Paths.get(
    "DataFiles", "file_wubba.txt");
  public static void main(String[] args) {
    BufferedReader source = null;
    try {
      source = new BufferedReader(
        new FileReader(thisFile.toFile()));
    } catch(FileNotFoundException e) {
      // 从文件打开错误中恢复
    }
    try {
      String first = source.readLine();
      eq(first, "wubba lubba dub dub");
    } catch(IOException e) {
      // 从读取错误中恢复
    }
    try {
      source.close();
    } catch(IOException e) {
      // 从关闭错误中恢复
    }
  }
}
```

上述代码中的每个操作都涉及到checked exceptions，必须将其放在try块中，否则Java会产生编译时错误。

在这个示例中，异常的处理方式似乎不太合理，但你仍然被迫编写try-catch块。

让我们在Kotlin中重写这个示例：

```kotlin
// interoperability/KotlinChecked.kt
import atomictest.eq
import java.io.File

fun main() {
  File("DataFiles/file_wubba.txt")
    .readLines()[0] eq
  "wubba lubba dub dub"
}
```

Kotlin允许我们将操作简化为一行代码，因为它为Java的`File`类添加了扩展函数。同时，Kotlin消除了checked exceptions。如果需要，我们可以在中间操作中使用try-catch块，但Kotlin不强制要求使用checked exceptions，这样就可以在不增加额外噪音代码的情况下提供错误报告。

Java库经常在程序员无法控制且通常无法恢复的情况下使用checked exceptions。在这些情况下，最好在顶层捕获异常并重新启动进程（如果可能）。要求所有中间层都传递异常只会增加理解代码时的认知负担。

如果你正在编写从Java调用的Kotlin代码，并且必须指定一个checked exception，Kotlin提供了`@Throws`注解，以便将此信息传递给Java调用者：

```kotlin
// interoperability/AnnotateThrows.kt
package interop
import java.io.IOException

@Throws(IOException::class)
fun hasCheckedException() {
  throw IOException

()
}
```

以下是如何从Java中调用`hasCheckedException()`的示例：

```java
// interoperability/CatchChecked.java
package interoperability;
import interop.AnnotateThrowsKt;
import java.io.IOException;
import static atomictest.AtomicTestKt.eq;

public class CatchChecked {
  public static void main(String[] args) {
    try {
      AnnotateThrowsKt.hasCheckedException();
    } catch(IOException e) {
      eq(e, "java.io.IOException");
    }
  }
}
```

如果你不处理异常，Java编译器会报错。

尽管Kotlin包括异常处理的语言支持，但它更注重错误报告，并将异常处理保留给那些在实际情况下可以从问题中恢复的情况（几乎只限于I/O操作）。

### Nullable类型和Java

Kotlin确保*纯粹的* Kotlin代码不会出现`null`错误，但是当你调用Java代码时，就没有这样的保证。在下面的Java代码中，`get()`有时会返回`null`：

```java
// interoperability/JTool.java
package interoperability;

public class JTool {
  public static JTool get(String s) {
    if(s == null) return null;
    return new JTool();
  }
  public String method() {
    return "Success";
  }
}
```

要在Kotlin中使用`JTool`，你必须了解`get()`的行为。在下面的示例中，我们提供了三种选择，分别对应于`a`、`b`和`c`的定义：

```kotlin
// interoperability/PlatformTypes.kt
package interop
import interoperability.JTool
import atomictest.eq

object KotlinCode {
  val a: JTool? = JTool.get("")  // [1]
  val b: JTool = JTool.get("")   // [2]
  val c = JTool.get("")          // [3]
}

fun main() {
  with(KotlinCode) {
    a?.method() eq "Success"     // [4]
    b.method() eq "Success"
    c.method() eq "Success"      // [5]
    ::a.returnType eq
      "interoperability.JTool?"
    ::b.returnType eq
      "interoperability.JTool"
    ::c.returnType eq
      "interoperability.JTool!"  // [6]
  }
}
```

- **[1]** 将类型指定为可为空。
- **[2]** 将类型指定为非空。
- **[3]** 使用类型推断。

`main()`中的`with()`允许我们在不使用`KotlinCode`限定的情况下引用`a`、`b`和`c`。由于这些标识符在`object`内部，我们可以使用成员引用语法和`returnType`属性来确定它们的类型。

要初始化`a`、`b`和`c`，我们向`get()`传递了一个非空字符串，因此`a`、`b`和`c`最终都具有非空引用，并且每个引用都可以成功调用`method()`。

- **[4]** 由于`a`可为空，因此在调

用成员函数时必须使用`?.`。
- **[5]** `c`的行为类似于非空引用，可以直接解引用而无需进行额外的检查。
- **[6]** 注意，`c`既不返回可空类型也不返回非空类型，而是返回完全不同的类型：`JTool!`。

`Type!`是Kotlin的*平台类型*，它没有表示法，你不能将其写入代码中。它用于Kotlin必须推断其域之外的类型时。

如果一个类型来自Java，访问它可能会产生空指针异常（NPE）。下面是当`JTool.get()`返回`null`引用时会发生的情况：

```kotlin
// interoperability/NPEOnPlatformType.kt
import interoperability.JTool
import atomictest.*

fun main() {
  val xn: JTool? = JTool.get(null)  // [1]
  xn?.method() eq null

  val yn = JTool.get(null)          // [2]
  yn?.method() eq null              // [3]
  capture {
    yn.method()                     // [4]
  } contains listOf("NullPointerException")

  capture {
    val zn: JTool = JTool.get(null) // [5]
  } eq "NullPointerException: " +
    "JTool.get(null) must not be null"
}
```

当你在Kotlin中调用像`JTool.get()`这样的Java方法时，其返回值（除非通过下一节中解释的注解进行了标注）是一个平台类型，在这种情况下是`JTool!`。

- **[1]** 由于`xn`是可为空类型`JTool?`，因此可以成功接收`null`。将值分配给可为空类型是安全的，因为Kotlin要求你在调用`method()`时使用`?.`进行`null`检查。
- **[2]** 在定义的时候，`yn`可以成功接收`null`而不会产生警告，因为Kotlin将其推断为平台类型`JTool!`。
- **[3]** 你可以使用安全访问调用`?.`来解引用`yn`，这种情况下它返回`null`。
- **[4]** 但是，使用`?.`并不是必需的。你可以直接解引用`yn`。在这种情况下，你会得到一个空指针异常，而没有任何有用的信息。
- **[5]** 分配给非空类型可能会导致空指针异常。Kotlin在赋值点检查null性。初始化`zn`失败，因为声明的类型`JTool`承诺`zn`不可为空，但它接收到了`null`，从而产生了空指针异常，这次带有一个有用的错误信息。

异常消息包含有关产生`null`的表达式的详细信息：`NullPointerException: JTool.get(null) must not be null`。即使它是一个运行时异常，但是全面的错误消息使问题比修复常规空指针异常更

容易。

总之，当在Kotlin中调用Java代码时，你需要考虑平台类型和可能的空指针异常。通过使用安全访问操作符`?.`并进行适当的空值检查，可以减少空指针异常的风险。

### 空值注解

如果你控制Java代码库，你可以为Java代码添加*空值注解*，从而避免微妙的空指针异常错误。`@Nullable`和`@NotNull`告诉Kotlin将Java类型视为可为空或不可为空。以下是我们在`JTool.java`中为Kotlin添加空值注解的示例：

```kotlin
// interoperability/AnnotatedJTool.java
package interoperability;
import org.jetbrains.annotations.NotNull;
import org.jetbrains.annotations.Nullable;

public class AnnotatedJTool {
  @Nullable
  public static JTool
  getUnsafe(@Nullable String s) {
    if(s == null) return null;
    return getSafe(s);
  }
  @NotNull
  public static JTool
  getSafe(@NotNull String s) {
    return new JTool();
  }
  public String method() {
    return "Success";
  }
}
```

在Java参数前应用注解仅影响该参数。在Java方法前应用注解则修改了返回类型。

当你在Kotlin中调用`getUnsafe()`和`getSafe()`时，Kotlin会将`AnnotatedJTool`成员函数视为本机Kotlin可空或非空类型：

```kotlin
// interoperability/AnnotatedJava.kt
package interop
import interoperability.AnnotatedJTool
import atomictest.eq

object KotlinCode2 {
  val a = AnnotatedJTool.getSafe("")
  // 不会编译通过：
  // val b = AnnotatedJTool.getSafe(null)
  val c = AnnotatedJTool.getUnsafe("")
  val d = AnnotatedJTool.getUnsafe(null)
}

fun main() {
  with(KotlinCode2) {
    ::a.returnType eq
      "interoperability.JTool"
    ::c.returnType eq
      "interoperability.JTool?"
    ::d.returnType eq
      "interoperability.JTool?"
  }
}
```

`@NotNull JTool`被转换为Kotlin的不可为空类型`JTool`，而带注解的`@Nullable JTool`则被转换为Kotlin的`JTool?`。你可以在`main()`中查看`a`、`c`和`d`的类型展示。

当期望一个非可空参数时，无法传递可为空的参数，即使它是用`@NotNull`注解的Java类型。因此，Kotlin不会编译通过`AnnotatedJTool.getSafe(null)`。

支持不同种类的空值注解，它们使用不同的名称：

- `@Nullable`和`@CheckForNull`是由JSR-305标准指定的。
- `@Nullable`和`@NonNull`在Android中使用。
- `@Nullable`和`@NotNull`由JetBrains工具支持。
- 还有其他的注解。你可以在Kotlin [文档](https://kotlinlang.org/docs/java-interop.html#nullability-annotations)中找到完整的列表。

Kotlin可以检测Java包或类的默认空值注解，这些注解在JSR-305标准中指定。如果默认情况下是`@NotNull`，则应该明确指定`@Nullable`注解。如果默认情况下是`@Nullable`，则应该明确指定`@NotNull`注解。[文档](https://kotlinlang.org/docs/java-interop.html#jsr-305-support)中包含选择默认注解的技术细节。

如果您开发混合Kotlin和Java的项目，使用Java代码中的空值注解可以使您的应用程序更安全。

### 集合与Java

这本书不需要Java知识。然而，当您在Java虚拟机（JVM）上编写Kotlin代码时，熟悉Java标准集合库会很有帮助，因为Kotlin使用它来创建自己的集合。

Java集合库是一组实现集合数据结构（如列表、集合和映射）的类和接口。这些数据结构通常具有清晰简单的接口，但为了提高速度可能有复杂的实现。

新的编程语言通常会从头开始创建自己的集合库。例如，Scala语言有自己的集合库，在许多方面超越了Java集合库，但也增加了在Scala和Java之间混合使用的挑战。

Kotlin的集合库有意地*没有*从头开始重新编写。相反，它在Java集合库的基础上进行了改进。例如，当您创建一个可变的`List`时，实际上是使用Java的`ArrayList`：

```kotlin
// interoperability/HiddenArrayList.kt
import atomictest.eq

fun main() {
  val list = mutableListOf(1, 2, 3)
  list.javaClass.name eq
    "java.util.ArrayList"
}
```
为了与Java代码无缝互操作，Kotlin使用Java标准库中的接口和常用实现。这带来了三个好处：

1. Kotlin代码可以轻松与Java代码混合使用。在将Kotlin集合传递给Java代码时，不需要进行额外的转换。
2. Java标准库中多年的性能调优对Kotlin程序员来说自动可用。
3. Kotlin应用程序附带的标准库很小，因为它使用Java集合而不是定义自己的集合。Kotlin标准库主要包含改进Java集合的扩展函数。

此外，Kotlin还修复了一个设计问题。在Java中，所有的集合接口都是可变的。例如，`java.util.List`具有修改列表的`add()`和`remove()`方法。正如本书中所展示的那样，可变性是许多编程问题的根源。因此，在Kotlin中，默认的`Collection`类型是只读的：

```kotlin
// interoperability/ReadOnlyByDefault.kt
package interop

data class Animal(val name: String)

interface Zoo {
  fun viewAnimals(): Collection<Animal>
}

fun visitZoo(zoo: Zoo) {
  val animals = zoo.viewAnimals()
  // Compile-time error:
  // animals.add(Animal("Grumpy Cat"))
}
```

只读集合更安全、更不容易出错，因为它们防止了意外修改。

Java提供了一种部分解决集合不可变性的方法：当返回一个集合时，你可以将其放在一个特殊的包装器中，对任何试图修改底层集合的操作抛出异常。这虽然不能产生静态类型检查，但仍然可以防止出现微妙的错误。然而，你必须记住在返回集合时进行包装，使其成为只读的，而在Kotlin中，你必须在想要一个可变集合时明确声明。

Kotlin有用于可变和只读集合的不同接口：

- `Collection`/`MutableCollection`
- `List`/`MutableList`
- `Set`/`MutableSet`
- `Map`/`MutableMap`

这些接口与Java标准库中的接口相对应：

- `java.util.Collection`
- `java.util.List`
- `java.util.Set`
- `java.util.Map`

在Kotlin中，与Java一样，`Collection`是`List`和`Set`的超类型。`MutableCollection`扩展自`Collection`，是`MutableList`和`MutableSet`的超类型。以下是基本结构的示例：

```kotlin
// interoperability/CollectionStructure.kt
package collectionstructure

interface Collection<E>
interface List<E>: Collection<E>
interface Set<E>: Collection<E>
interface Map<K, V>
interface MutableCollection<E>
interface MutableList<E>:
  List<E>, MutableCollection<E>
interface MutableSet<E>:
  Set<E>, MutableCollection<E>
interface MutableMap<K, V>: Map<K, V>
```

为简单起见，我们只显示了Kotlin标准库中的名称，而没有显示完整的声明。

Kotlin的可变集合与其Java对应物相匹配。如果你将`kotlin.collections`中的`MutableCollection`与`java.util.List`进行比较，你会发现它们声明了相同的成员函数（在Java术语中称为*方法*）。Kotlin的`Collection`、`List`、`Set`和`Map`也复制了Java的接口，但没有包含任何修改方法。

`kotlin.collections.List`和`kotlin.collections.MutableList`都可以从Java中看到，它们被视为`java.util.List`。这些接口是特殊的：它们只存在于Kotlin中，但在字节码级别上，它们都被替换为Java的`List`。

Kotlin的`List`可以转换为Java的`List`：

```kotlin
// interoperability/JavaList.kt
import atomictest.eq

fun main() {
  val list = listOf(1, 2, 3)
  (list is java.util.List<*>) eq true
}
```

这段代码会产生一个警告：

- *在Kotlin中不应使用这个类。*
- *请使用kotlin.collections.List或kotlin.collections.MutableList代替。*

这是提醒在Kotlin编程时要使用Kotlin的接口而不是Java的接口。

请记住，只读不等同于不可变。通过只读引用，无法修改集合，但是它仍然可以被改变：

```kotlin
// interoperability/ReadOnlyCollections.kt
import atomictest.eq

fun main() {
  val mutable = mutableListOf(1, 2, 3)
  // 只读引用指向可变列表：
  val list: List<Int> = mutable
  mutable += 4
  // list已经改变：
  list eq "[1, 2, 3, 4]"
}
```

在这个例子中，只读的`list`引用了一个`MutableList`，然后通过操作`mutable`对其进行修改。因为所有的Java集合都是可变的，所以Java代码可以修改只读的Kotlin集合，即使你通过只读引用传递给它。

Kotlin集合并不能完全确保安全性，但在提供更好的库和保持与Java的兼容性之间提供了一个良好的平衡。

### Java基本类型

在Kotlin中，你调用构造函数来创建一个对象，但在Java中，你必须使用`new`关键字来生成一个对象。`new`会将结果对象放在堆上。这些类型被称为*引用类型*。

对于基本类型（如数字），在堆上创建对象可能效率较低。对于这些类型，Java采用了C和C++的方法：不使用`new`关键字创建变量，而是创建一个非引用的“自动”变量，直接保存该值。自动变量被放置在栈上，使其更高效。这些类型在JVM上得到了特殊处理，被称为*原始类型*。

原始类型有固定的数量：`boolean`、`int`、`long`、`char`、`byte`、`short`、`float`和`double`。原始类型总是包含非`null`值，不能用作泛型参数。如果需要存储`null`或将这些类型用作泛型参数，可以使用Java标准库中定义的相应引用类型，如`java.lang.Boolean`或`java.lang.Integer`。这些类型通常被称为*包装类型*或*装箱类型*，以强调它们仅包装原始值并将其存储在堆上。

```java
// interoperability/JavaWrapper.java
package interoperability;
import java.util.*;

public class JavaWrapper {
  public static void main(String[] args) {
    // 原始类型
    int i = 10;
    // 包装类型
    Integer iOrNull = null;
    List<Integer> list = new ArrayList<>();
  }
}
```

Java区分引用类型和原始类型，但Kotlin不区分。你在定义整数的`var`/`val`或在使用它作为泛型参数时使用相同的`Int`类型。在JVM级别，Kotlin使用相同的原始类型支持。在生成字节码时，Kotlin会尽可能地将`Int`替换为原始类型`int`。只有使用包装类型才能表示可空的`Int?`或用作泛型参数的`Int`：

```kotlin
// interoperability/KotlinWrapper.kt
package interop

fun main() {
  // 生成原始类型int：
  val i = 10
  // 生成包装类型：
  val iOrNull: Int? = null
  val list: List<Int> = listOf(1, 2, 3)
}
```

通常情况下，你不需要过多考虑Kotlin编译器生成的是原始类型还是包装类型，但了解它在JVM上的实现方式是有用的。

---

[文档](https://kotlinlang.org/docs/java-interop.html)详细解释了Kotlin/Java互操作性的细微差别。