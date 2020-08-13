---
date: 2020-08-10
title: "从HelloWold开始，深入浅出C++ 20 Coroutine TS"
linkTitle: "C++ 20 Coroutine"
description: "分析C++20协程的使用和原理分析"
author: 顾起威 ([@gqw](https://gitee.com/gqw))
description: >
  分析研究协程原理
---

## 缘起

前一阵子在查看`asio`库的时候看到在example目录中已经提供了`coroutines_ts`代码。很是好奇，便慢慢翻看代码，在感叹Coroutine给异步编程带来的优雅简洁的同时也带来了许多概念上的复杂和难以理解。由于C++ Coroutine还是一个比较新的概念，各个编译器厂商的实现还处于实验性的阶段<sup>[[1]](#ref_1)</sup> ，网上的资料少而松散。由此内心涌出一个念头想把自己的学习过程记录下来以期帮助后面学习的人。

## 从`HelloWorld`说起

再过两年这个世界的第一条`HelloWorld`代码就要有50年历史了<sup>[[2]](#ref_2)</sup> ，在此先提前蹭下热点，:smile:...

```cpp
#include <iostream>
int main() { 
  std::cout << "hello, world" << std::endl; 
  return 0;
}
```

这是一段平淡无奇的符合`C++98`标准的代码，你甚至可以用古董级编译器`GCC 4.3`都能编译通过<sup>[[3]](#ref_3)</sup>。但是我希望通过对这段代码的演化带领大家学习了解C++ 20中的Coroutine。

这段代码的作用很简单，打印一条"hello, world"，执行的也很快。但是这个世界并不总是那么简单美好，如果打印的逻辑非常耗时（特别是读取磁盘文件和进行网络请求时）而又需要频繁的调用就有些尴尬了，好了，我们改下代码来模拟这种情形：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std::chrono_literals;

void remote_query() {
  // 假设这是个远程网络请求
  std::this_thread::sleep_for(4s);
  std::cout << "hello, world" << std::endl;
}

int main() { 
  for (;;) {
    remote_query();
  }
  return 0;
}
```

改过后的代码不停的进行耗时的远程调用，每次远程调用需要消耗4秒的时间才能返回。可以看到，每次进行调用时程序都要暂停住（也就是卡住）而做不了其它事情。如果这是个带`UI`(用户界面)的程序，每次进行调用界面就要卡住，用户肯定是不能忍受的。

---
**注意**

上面的代码由于使用了`thread`<sup>[[4]](#ref_4)</sup>(C++ 11)和`chrono_literals`<sup>[[5]](#ref_5)</sup>(C++ 14)，所以编译器需要支持C++ 14标准。

---

## 多线程+`Callback`解决方案

为了解决上面的问题，我们对上面的代码进行如下改造：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std::chrono_literals;
#include <string>
#include <future>

void remote_query(uint32_t query_index, std::function<void(uint32_t)>&& callback) { 
 std::thread t(
      [query_index, callback = std::move(callback)]() {
        std::this_thread::sleep_for(4s);
        std::cout << std::to_string(query_index) + " times query: Hello, world!" << std::endl; 
        callback(query_index);
      });
  t.detach();
}

void remote_callback(uint32_t query_index) {
 remote_query(++query_index, remote_callback);
}

int main() {
  remote_query(1, remote_callback);
  while (true)
  {
    std::this_thread::sleep_for(1s);
    std::cout << "main thread doing other things..." << std::endl;
  }
  return 0;
}
```
打印结果：
```
main thread doing other things...
main thread doing other things...
main thread doing other things...
main thread doing other things...
1 times query: Hello, world!
main thread doing other things...
main thread doing other things...
main thread doing other things...
main thread doing other things...
2 times query: Hello, world!
main thread doing other things...
main thread doing other things...
```

是的，我们可以通过多线程解决卡顿的问题，但这是个好的方案吗？不说线程的开销（其实创建和销毁一个线程无论从时间还是空间上来说都是非常大的开销），单从代码的组织来看也足够让人头疼的了。上面的代码只是循环调用同样的远程方法，如果`remote_callback`调用`remote_query1`(另一个远程调用)，`remote_query1`回调中再调用`remote_query2`...如此下去代码逻辑将会非常恐怖。为什么说是恐怖，因为这完全不符合人类的思维逻辑。人类喜欢的代码是有条不紊的按步执行，而不是在各个回调中跳来跳去。最好就像前面的同步代码，一个请求走完再执行下一个请求。

## 主角登场

有没有什么方法解决这种复杂性呢？有的，技术牛人们早已发现这个问题并且给出了设计模型，有类似于`libevent`的reactor模式<sup>[[6]](#ref_6)</sup>，也有`asio`的`proactor`模式<sup>[[7]](#ref_7)</sup>。但`Reactor`，`Proactor`模式的核心思想都是通过事先注册回调函数，收到消息再根据消息内容查找并执行回调，有效的将层层包裹的回调给摊平以达到降低回调函数管理的复杂度。但是无论`Reactor`还是`Proactor`调用逻辑和回调逻辑还是被硬生生的给割裂开了。如何弥合这里的割裂呢？这就需要请出我们今天的主角`Corouter`了，还是先看代码吧：

```cpp
#include <iostream>
#include <thread>
#include <chrono>
using namespace std::chrono_literals;
#include <string>
#include <future>

std::future<std::string> remote_query(uint32_t query_index) {
  return std::async([query_index]() {
    std::this_thread::sleep_for(4s);
    return std::to_string(query_index) + " times query: Hello, world!";
  });
}

std::future<void> remote_query_all() { 
  uint32_t query_index  = 0;
  for (;;)
  {
    std::string ret = co_await remote_query(++query_index);
    std::cout << ret << std::endl;
  }
}

int main() {
  remote_query_all();

  while (true)
  {
    std::this_thread::sleep_for(1s);
    std::cout << "main thread doing other things..." << std::endl;
  }
  return 0;
}
```
打印结果：
```
main thread doing other things...
main thread doing other things...
main thread doing other things...
1 times query: Hello, world!
main thread doing other things...
main thread doing other things...
main thread doing other things...
main thread doing other things...
2 times query: Hello, world!
main thread doing other things...
main thread doing other things...
main thread doing other things...
```

看到了吗？打印结果还是一样的，但是回调没有了,  从代码上看函数`remote_query_all`的逻辑也是在一个for循环中按步执行。好了，讲到这里只是希望你能直观的体会到使用Coroutine的作用和好处，但是你心中一定会有n个问号。

- 这不科学呀，`remote_query`返回值不是`std::future<std::string>`吗？ 怎么直接赋值给`std::string`了？
- `co_await` 有时什么鬼？
- `remote_query` 是个异步调用,  返回的`ret`直接用会不会崩溃？
- ...

请少安毋躁， 暂时按下心中的疑惑，因为这边的水很深，坑很大，不是三言两语能够解释的清楚的。我会尽量努力在后面的内容中消除你心中的困惑。



## 先从定义说起

翻开`cppreference`看看官方的定义<sup>[[8]](#ref_8)</sup>：

```
协程是能暂停执行以在之后恢复的函数。协程是无栈的：它们通过返回到调用方暂停执行，并且从栈分离存储恢复所要求的数据。这允许编写异步执行的顺序代码（例如不使用显式的回调来处理非阻塞 I/O），还支持对惰性计算的无限序列上的算法及其他用途。

若函数的定义做下列任何内容之一，则它是协程：

. 用 co_await 运算符暂停执行，直至恢复
. 用关键词 co_yield 暂停执行并返回一个值
. 用关键词 co_return 完成执行并返回一个值
```

读完之后似乎还是什么都不知道，好吧，现在我们试着逐句翻译成“白话文”。

- 协程是能暂停执行以在之后恢复的函数。

  这里的协程就是前面说的`Coroutine`, 这句说出了协程的本质。首先它是一个函数，然后这个函数与普通函数的区别是能够“暂停”和“恢复”执行代码。所以协程最关键的点是“暂停”和“恢复”。用一段代码说明下：

  ```
  void func() {
  	code1;
  	code2;
  	code3;
  }
  ```

  



## 参考引用

[1]  <a id="ref_1" href="https://en.cppreference.com/w/cpp/compiler_support" >C++ compiler support</a>`[EB/OL].`  https://en.cppreference.com/w/cpp/compiler_support

[2]  <a id="ref_2" href="https://en.wikipedia.org/wiki/%22Hello,_World!%22_program" >"Hello, World!" program</a>`[EB/OL].`https://en.wikipedia.org/wiki/%22Hello,_World!%22_program

[3]  <a id="ref_3" href="https://gcc.gnu.org/projects/cxx-status.html#cxx98" >C++ Standards Support in GCC</a>`[EB/OL].`https://gcc.gnu.org/projects/cxx-status.html#cxx98

[4]  <a id="ref_4" href="https://en.cppreference.com/w/cpp/thread/get_id" >std::this_thread::get_id</a>`[EB/OL].`https://en.cppreference.com/w/cpp/thread/get_id

[5]  <a id="ref_5" href="https://en.cppreference.com/w/cpp/language/user_literal%23Standard_library" >User-defined literals</a>`[EB/OL].`https://en.cppreference.com/w/cpp/language/user_literal%23Standard_library

[6]  <a id="ref_6" href="https://aceld.gitbooks.io/libevent/content/31_reactorfan_ying_dui_mo_shi.html" >reactor反应堆模式</a>`[EB/OL].`https://aceld.gitbooks.io/libevent/content/31_reactorfan_ying_dui_mo_shi.html

[7]  <a id="ref_7" href="https://www.boost.org/doc/libs/1_47_0/doc/html/boost_asio/overview/core/async.html" >Proactor and Boost.Asio</a>`[EB/OL].`https://www.boost.org/doc/libs/1_47_0/doc/html/boost_asio/overview/core/async.html

[8]  <a id="ref_8" href="https://en.cppreference.com/w/cpp/language/coroutines" >Coroutines (C++20)
</a>`[EB/OL].`https://en.cppreference.com/w/cpp/language/coroutines



