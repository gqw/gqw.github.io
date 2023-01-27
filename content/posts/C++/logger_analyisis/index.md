---
date: 2023-01-18
title: "日志系统性能分析与优化"

menu:
  sidebar:
    name: "日志系统性能分析与优化"
    identifier: 13b8c649-a7a6-4d91-b892-68d0dacda868
    parent: C++
    weight: 11
hero: log_system.png
---


日志系统在任何稍微有点规模的软件系统中都是必不可少的组成部分，它是我们分析代码和解决问题的有效手段和得力助手。一份详尽而精准的日志能让我们解决问题事半功倍。很多时候由于日志的频繁打印很容易造成日志系统本身成为性能热点，下面让我们详细分析下作为日志系统的调用方需要注意的一些问题。

## 日志打印

大多数的日志系统都会提供`print`和`stream`两种调用方式。简单的如：

```cpp
logger::get().info("%s", "log message");
logger::get().info() << "log message" << std::endl;
```

## 使用日志等级进行输出优化

除此之外，日志系统一般还会提供日志分级功能，通过日志分级我们可以在程序运行时动态控制日志输出，减少不必要的日志打印。日志分级可以通过如下伪代码试下：

```cpp
class Logger {
public:
    void trace(const std::string_view& fmt, ...) {
        if (log_level_ < 0)   // 通过日志等级，判断是否需要真正打印
            return;

        printf(...);
    }

    void debug(const std::string_view& fmt, ...) {
        if (log_level_ < 1)
            return;

        printf(...);
    }

    void info(const std::string_view& fmt, ...) {
        if (log_level_ < 2)
            return;

        printf(...);
    }

    void set_level(int level) {
      log_level_ = level;
    }

    ...

private:
    int log_level_ = 0;
};
```

通过上面的代码我们可以看到如果调用`set_level(2)` 那么下面三条日志只打印一条日志结果：

```cpp
  logger::get().set_level(2);
  logger::get().trace("%s", "hello, I'm trace log");
  logger::get().debug("%s", "hello, I'm debug log");
  logger::get().info("%s", "hello, I'm info log");
```

```
hello, I'm info log
```

`trace` 和 `debug` 日志被 `log_level_` 开关优化过滤掉了。在实际的工程实践中我们除了`trace`、`debug`、`info` 日志级别外通常还有 `warnning`、`error`、`fatal`等级别。

```
trace:  尽可能详细的日志，甚至代码的每个逻辑分支都可以打印一条，trace日志能够帮助了解系统的运行流程
debug： 用于帮助调试的日志
info：  少量但关键信息的日志，一般用于打印系统的关键状态
warn：  警告日志，代表系统逻辑有异常，但是不影响系统的正常运作。警告日志不代表可以放任不管，出现警告日志一般代表了超出了预期的逻辑存在，虽然现在能够运行，但不代表以后不会出现致命问题，出现警告日志最好能仔细排查原因，如果确定没有问题，可以去掉此日志
error： 代表了出现了明确的错误逻辑，需要认真对待，仔细排查
fatal： 代表出现了致命错误，如果出现此日志信息意味着程序应该立即准备善后工作，退出进程
```

工程如果处于开发调试阶段一般会将日志级别定在`info`级别或以下(即打开`debug`或`trace`)，但线上运行时会将日志级别调到`info`级别或以上，最起码需要保证`error`和`fatal`的输出。建议设为`info`级别，因为`info`日志本身就比较少，不会成为性能瓶颈，而`warn`日志可以帮助我们发现和优化潜在的问题。

## 使用宏进一步优化日志

虽然通过日志级别减少了日志输出，但是却没有减少日志调用。什么意思呢？我们下面几个例子的反汇编代码来直观的了解下原理。

首先让我们创建一个空的工程，这里面只有一个`main`函数，然后我们通过一步步添加代码来观察比较反汇编代码的变化。

```cpp
int main() {
    return 0;
}
```

得到汇编指令:

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     eax, 0
        pop     rbp
        ret
```

现在让我们添加个简单的日志系统：

```cpp
#include <stdio.h>
#include <stdarg.h>

class Logger  {
public:
    static void trace(const char* fmt, const char* msg) {
        if (log_level_ < 0)   // 通过日志等级，判断是否需要真正打印
            return;

        printf(fmt, msg);
    }

    static void debug(const char* fmt, const char* msg) {
        if (log_level_ < 1)
            return;

        printf(fmt, msg);
    }

    ...

    static void set_level(int level) {
      log_level_ = level;
    }

private:
    static int log_level_;
};

int main() {
    Logger::set_level(2);
    Logger::trace("%s", "trace log");
    Logger::debug("%s", "debug log");
    Logger::info("%s", "info log");
    return 0;
}
```


让我们看下`main`函数的反汇编代码:
```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     edi, 2
        call    Logger::set_level(int)
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, OFFSET FLAT:.LC1
        call    Logger::trace(char const*, char const*)
        mov     esi, OFFSET FLAT:.LC2
        mov     edi, OFFSET FLAT:.LC1
        call    Logger::debug(char const*, char const*)
        mov     esi, OFFSET FLAT:.LC3
        mov     edi, OFFSET FLAT:.LC1
        call    Logger::info(char const*, char const*)
        mov     eax, 0
        pop     rbp
        ret
```
我们可以看到虽然我们只打印了：
```
info log
```
一行日志， 但是我们还是看到了`trace`和`debug`的代码调用，在`trace`和`debug`的函数内部条件判断的指令还是会被执行。在上面的代码中我们已经尽量地简化，对于一个复杂的日志系统会产生更多的调用指令，如果我们的`trace`和`debug`的日志非常多，这必然会造成巨大的性能浪费。

在C++中我们为了避免这样的性能开销一般会使用宏来控制无效指令的生成, 例如：

```cpp
#define LOG_LEVEL_TRACE    0
#define LOG_LEVEL_DEBUG    1
#define LOG_LEVEL_INFO     2
#define LOG_LEVEL_WARN     3
#define LOG_LEVEL_ERROR    4
#define LOG_LEVEL_FATAL    5
#define LOG_LEVEL_CLOSE    6

int main() {
    Logger::set_level(2);
#if LOGGER_LEVEL <= LOG_LEVEL_TRACE
    Logger::trace("%s", "trace log");
#endif
#if LOGGER_LEVEL <= LOG_LEVEL_DEBUG
    Logger::debug("%s", "debug log");
#endif
#if LOGGER_LEVEL <= LOG_LEVEL_INFO
    Logger::info("%s", "info log");
#endif
    return 0;
}
```

对应的汇编指令如下：

```asm
main:
        push    rbp
        mov     rbp, rsp
        mov     edi, 2
        call    Logger::set_level(int)
        mov     esi, OFFSET FLAT:.LC0
        mov     edi, OFFSET FLAT:.LC1
        call    Logger::info(char const*, char const*)
        mov     eax, 0
        pop     rbp
        ret
```

可以看到已经没有`trace`和`debug`的调用指令了， 但在每个调用的地方都这样用宏包裹下实在是太麻烦，更一般的做法是像下面这样将日志调用做成宏的样子：

```cpp
#if (LOGGER_LEVEL > -1)
#	define	 LOG_TRACE(fmt, ...) Logger::trace("TRACE "##fmt, #__VA_ARGS__)
#else
#	define	 LOG_TRACE(fmt, ...)
#endif

#if (LOGGER_LEVEL > 0)
#	define	 LOG_DEBUG(fmt, ...) Logger::debug("DEBUG "##fmt, #__VA_ARGS__)
#else
#	define	 LOG_DEBUG(fmt, ...)
#endif

#if (LOGGER_LEVEL > 1)
#	define	 LOG_INFO(fmt, ...) Logger::info("INFO "##fmt, #__VA_ARGS__)
#else
#	define	 LOG_INFO(fmt, ...)
#endif

...


int main() {
    Logger::set_level(2);
    LOG_TRACE("%s", "trace log");
    LOG_DEBUG("%s", "debug log");
    LOG_INFO("%s", "info log");
    return 0;
}
```

## 应不应该使用流式日志输出

大多数的日志系统除了提供`print`日志输出方式，一般还会提供流式输出方式，例如：

```cpp
Logger::get().trace() << "stream trace: " << 100;
Logger::get().debug() << "stream debug: " << 100;
Logger::get().info() << "stream info: " << 100;
```

这种方式，极大的方便了日志使用者，但也造成了性能的退化。为什么这么说呢？让我们来看个简单的例子：

```cpp
std::cout << " stream log" << 100 << " test";
```

我们看下反汇编代码(为了方便观察去掉了模板参数信息)：

```asm
    mov     esi, OFFSET FLAT:.LC0
    mov     edi, OFFSET FLAT:_ZSt4cout
    call    std::basic_ostream<>& std::operator<< (std::basic_ostream<>&, char const*)
    mov     esi, 100
    mov     rdi, rax
    call    std::basic_ostream<>::operator<<(int)
    mov     esi, OFFSET FLAT:.LC1
    mov     rdi, rax
    call    std::basic_ostream<>& std::operator<< (std::basic_ostream<>&, char const*)
```

从上面的汇编代码中我们可以看到每一个`<<`操作符就是一次函数调用（call 指令）。我们无法实现像前面介绍的`LOG_XXXX`方式定义宏来在编译期优化掉无效的日志调用。

那是否就束手无策了呢？其实我们还是有一些技巧来优化这种开销的，首先我们需要定义一个支持流操作的日志类:

```cpp
class LoggerOfNone {
public:
	LoggerOfNone() = default;

    template<typename T>
	LoggerOfNone& operator<<(T content) {
		return *this;
	}
};
```

可以看到`operator<<`操作符函数没有做任何事，相当于是一个空调用，然后我们进行如下调用：

```cpp
int main() {
    LoggerOfNone() << " stream log" << 100 << " test";
    return 0;
}
```

我们设置`g++`优化选项为`-O0`,即不做任何优化，得到反汇编代码：


```asm
main:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        lea     rax, [rbp-1]
        mov     esi, OFFSET FLAT:.LC0
        mov     rdi, rax
        call    LoggerOfNone& LoggerOfNone::operator<< <char const*>(char const*)
        mov     esi, 100
        mov     rdi, rax
        call    LoggerOfNone& LoggerOfNone::operator<< <int>(int)
        mov     esi, OFFSET FLAT:.LC1
        mov     rdi, rax
        call    LoggerOfNone& LoggerOfNone::operator<< <char const*>(char const*)
        mov     eax, 0
        leave
        ret
```

可以看到仍然是三次`operator<<`函数调用， 但是不用泄气，我们现在将优化选项设为`-O3`，我们再试一次得到：

```asm
main:
        xor     eax, eax
        ret
```

神奇的事情发生了，我们得到了非常干净的结果，没有任何多余的调用发生。这是因为我们调用一个空函数时编译器会自动优化掉。所以我们可以根据这个特性可以做如下优化：

```cpp

LoggerOfNone g_logger_none;

#if (LOGGER_LEVEL > -1)
#	define	 LOG_TRACE(fmt, ...) Logger::trace("TRACE "##fmt, #__VA_ARGS__)
#	define	 STREAM_TRACE() std::cout
#else
#	define	 LOG_TRACE(fmt, ...)
#	define	 STREAM_TRACE() g_logger_none
#endif

...

```
可以看到当不满足日志等级时，我们边返回`LoggerOfNone`空日志对象。这样起码能保证在有编译器优化的情况下能得到较好的结果。

但是是不是这样就能高枕无忧了呢？很不幸，并不能！！！ 让我们看下下面的例子：

```cpp
std::string GetTestStr() {
    std::cout << "tt" << std::endl;
    return "hello";
}

int main() {
    STREAM_TRACE() << " stream log" << 100 << GetTestStr();
    return 0;
}
```

优化选项仍然设为`-O3`，得到：

```
main:
        sub     rsp, 40
        mov     rdi, rsp
        call    GetTestStr[abi:cxx11]()
        mov     rdi, QWORD PTR [rsp]
        lea     rax, [rsp+16]
        cmp     rdi, rax
        je      .L12
        call    operator delete(void*)
.L12:
        xor     eax, eax
        add     rsp, 40
        ret
```

我们看到虽然`operator<<`操作符函数调用被优化了， 但是`GetTestStr()`却被调用了，实际上我们并不想它被调用。

所以从性能上考虑`stream`与`print`方式相比， 我这里更推荐使用`print`方式输出。

## 另一种选择

从前面分析我们可以得到`print`日志输出方式能得到更好的性能的结论，但是`print`格式化打印对于使用者来说实在是太不友好了，可以说相当难用。但是对于Mordern C++来说我们有了另一种选择，那就是[`std::format`](https://en.cppreference.com/w/cpp/utility/format/format)(需要C++ 20支持)也可以使用[`fmt`](https://fmt.dev/latest/index.html)（只需要C++ 11）。

最后推荐[`spdlog`](https://github.com/gabime/spdlog)日志系统， 它能够完美支持前面所说的三种调用方式。通过简单的封装可以实现如下效果：

```cpp
     logger::get().set_level(spdlog::level::info);
     STM_DEBUG() << "STM_DEBUG " << 2;
     PRINT_WARN("PRINT_WARN, %d", 2);
     LOG_INFO("LOG_INFO {}", 2);
```

详细代码可以参考[`spdlog_wrapper`](https://github.com/gqw/spdlog_wrapper)







