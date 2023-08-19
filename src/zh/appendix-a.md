# 附录 A：AtomicTest

> 这个简洁的测试框架用于验证书中的例子。它还有助于在学习过程中早期引入和推广单元测试。

该框架在以下原子中进行了描述：

- [测试](./se02-ch07.md) 介绍了该框架，并描述了 `eq` 和 `neq` 函数以及 `trace` 对象。
- [异常](./se02-ch08.md) 介绍了 `capture()` 函数。
- [异常处理](./se06-ch01.md) 描述了 `capture()` 函数的实现。
- [单元测试](./se06-ch06.md) 使用 AtomicTest 来帮助引入单元测试的概念。

```kotlin
// AtomicTest/AtomicTest.kt
package atomictest
import kotlin.math.abs
import kotlin.reflect.KClass

const val ERROR_TAG = "[Error]: "

private fun <L, R> test(
  actual: L,
  expected: R,
  checkEquals: Boolean = true,
  predicate: () -> Boolean
) {
  println(actual)
  if (!predicate()) {
    print(ERROR_TAG)
    println("$actual " +
      (if (checkEquals) "!=" else "==") +
      " $expected")
  }
}

/**
 * Compares the string representation
 * of this object with the string `rval`.
 */
infix fun Any.eq(rval: String) {
  test(this, rval) {
    toString().trim() == rval.trimIndent()
  }
}

/**
 * Verifies this object is equal to `rval`.
 */
infix fun <T> T.eq(rval: T) {
  test(this, rval) {
    this == rval
  }
}

/**
 * Verifies this object is != `rval`.
 */
infix fun <T> T.neq(rval: T) {
  test(this, rval, checkEquals = false) {
    this != rval
  }
}

/**
 * Verifies that a `Double` number is equal
 * to `rval` within a positive delta.
 */
infix fun Double.eq(rval: Double) {
  test(this, rval) {
    abs(this - rval) < 0.0000001
  }
}

/**
 * Holds captured exception information:
 */
class CapturedException(
  private val exceptionClass: KClass<*>?,
  private val actualMessage: String
) {
  private val fullMessage: String
    get() {
      val className =
        exceptionClass?.simpleName ?: ""
      return className + actualMessage
    }
  infix fun eq(message: String) {
    fullMessage eq message
  }
  infix fun contains(parts: List<String>) {
    if (parts.any { it !in fullMessage }) {
      print(ERROR_TAG)
      println("Actual message: $fullMessage")
      println("Expected parts: $parts")
    }
  }
  override fun toString() = fullMessage
}

/**
 * Captures an exception and produces
 * information about it. Usage:
 *    capture {
 *      // Code that fails
 *    } eq "FailureException: message"
 */
fun capture(f:() -> Unit): CapturedException =
  try {
    f()
    CapturedException(null,
      "$ERROR_TAG Expected an exception")
  } catch (e: Throwable) {
    CapturedException(e::class,
      (e.message?.let { ": $it" } ?: ""))
  }

/**
 * Accumulates output when called as in:
 *   trace("info")
 *   trace(object)
 * Later compares accumulated to expected:
 *   trace eq "expected output"
 */
object trace {
  private val trc = mutableListOf<String>()
  operator fun invoke(obj: Any?) {
    trc += obj.toString()
  }
  /**
   * Compares trc contents to a multiline
   * `String` by ignoring white space.
   */
  infix fun eq(multiline: String) {
    val trace = trc.joinToString("\n")
    val expected = multiline.trimIndent()
      .replace("\n", " ")
    test(trace, multiline) {
      trace.replace("\n", " ") == expected
    }
    trc.clear()
  }
}
```