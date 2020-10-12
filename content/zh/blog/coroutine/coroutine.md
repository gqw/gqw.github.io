---
date: 2020-10-01
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

再过两年这个世界的第一条`HelloWorld`代码就要有50年历史了<sup>[[2]](#ref_2)</sup> ，在此先提前蹭下热点，:)...

```cpp
#include <iostream>
int main() { 
  std::cout << "hello, world" << std::endl; 
  return 0;
}
```
<center><div style="color:orange; display: inline-block; color: #999; padding: 2px;">代码1</div></center></br>


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
<center><div style="color:orange; display: inline-block; color: #999; padding: 2px;">代码2</div></center></br>

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
<center><div style="color:orange; display: inline-block; color: #999; padding: 2px;">代码3</div></center></br>

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

是的，我们可以通过多线程解决卡顿的问题，但这是个好的方案吗？不说线程的开销（其实创建和销毁一个线程无论从时间还是空间上来说都是非常大的开销），单从代码的组织来看也足够让人头疼的了。上面的代码只是循环调用同样的远程方法，如果`remote_callback`调用`remote_query1`(另一个远程调用)，`remote_query1`回调中再调用`remote_query2`...如此下去代码逻辑将会非常恐怖。如果不能明白我说的是什么意思，`callback hell` 了解下:)， 为什么说是恐怖，因为这完全不符合人类的思维逻辑。人类喜欢的代码是有条不紊的按步执行，而不是在各个回调中跳来跳去。最好就像前面的同步代码，一个请求走完再执行下一个请求。

## 主角登场

有没有什么方法解决这种复杂性呢？有的，技术牛人们早已发现这个问题并且给出了设计模型，有类似于`libevent`的`Reactor`模式<sup>[[6]](#ref_6)</sup>，也有`asio`的`Proactor`模式<sup>[[7]](#ref_7)</sup>。但`Reactor`，`Proactor`模式的核心思想都是通过事先注册回调函数，收到消息再根据消息内容查找并执行回调，有效的将层层包裹的回调给摊平以达到降低回调函数管理的复杂度。但是无论`Reactor`还是`Proactor`调用逻辑和回调逻辑还是被硬生生的给割裂开了。如何弥合这里的割裂呢？这就需要请出我们今天的主角`Corouter`了，还是先看代码吧：

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
<center><div style="color:orange; display: inline-block; color: #999; padding: 2px;">代码4</div></center></br>

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
- `co_await` 又是什么鬼？
- `remote_query` 是个异步调用,  返回的`ret`直接用会不会崩溃？
- ...

请少安毋躁， 暂时按下心中的疑惑，因为这边的水很深，坑很大，不是三言两语能够解释的清楚的。我会尽量努力在后面的内容中消除你心中的困惑。



## 先从定义说起

翻开`cppreference`看看协程的定义<sup>[[8]](#ref_8)</sup>：

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
    void foo() {
    	code1;
    	code2;
    	code3;
    }
    
   int main() {
    	foo();
    	return 0;
    }
  ```
  <center><div style="color:orange; display: inline-block; color: #999; padding: 2px;">代码5</div></center></br>
  
  这是一段普通的代码，代码从main函数开始，先调用`foo`函数，接着依次调用`code1`、`code2`、`code3`，然后返回`main`函数，最后执行`return 0`。这种执行顺序是我们见到的最常见的顺序，代码逐行执行，按部就班符合我们的预期。但是如果是协程，代码就可以在执行完`code1`后立马返回到main函数，在适当时候再返回`foo`执行`code1`后面的代码`code2`——这就是前面所谓的“暂停”和“恢复”。其实仔细想想所谓协程也只是计算机执行顺序的一种规范，有点类似goto指令，都是用于改变常规的代码执行顺序，只是这些规范我们用的比较少见而已。其实翻阅维基百科的“协程”页面<sup>[[9]](#ref_9)</sup>，你会发现其实协程历史并不短，起码已经超过半个世纪了。但是直到最近几年才开始火热是有原因的，就像goto语句一样，过多的自由有时也是一种灾难（代码逻辑混乱，难以维护）。但是随着计算机硬件的发展（多核多线程）和一些成熟的案例（如腾讯的`libco`<sup>[[10]](#ref_10)</sup>和`golang`<sup>[[11]](#ref_11)</sup>）出现，让人们认识到了协程的优势和闪光点。

- 协程是无栈的。

  “无栈” 两个字隐藏了太多的概念，对于一个完全没有接触过协程的人来说这个概念太含糊和陌生了。这里的“无栈”是相对于协程的另一种实现方式“有栈”协程而言的。在介绍这两个概念前先看下普通函数调用堆栈。函数堆栈非正式的理解就是函数调用时申请的用于存放参数和临时变量的内存块，按照后进先出（LIFO, Last In First Out）的规则分配内存，因为只有push和pop两种操作所以比较简单和快捷，由编译器自动管理其分配内存的生命周期。 在https://godbolt.org/里写下测试代码得到如下汇编指令：
  

  <center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://i.loli.net/2020/10/02/g2thx7oSOYybrpE.png">
    <br>
    <div style="color:orange; border-bottom: 0px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图1 函数调用汇编分析源码</div>
  </center>

  经过简单整理可得到堆栈的使用情况如下图所示：

  <center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://i.loli.net/2020/10/02/zdcHC9wYpSAfrv3.png">
    <br>
    <div style="color:orange; border-bottom: 0px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图2 函数调用堆栈变化</div>
  </center>

  ---
  **注意**

  上面的代码是使用`x64`位的`msvc`编译的，和经常见到`x86`版本相比64位版本的更加简洁易懂，少了“基址指针寄存器”的概念所以省去了保存和恢复基址指针的操作。这里也没有常规的push和pop操作，而是直接通过sub和add相应的地址偏移获得相同的效果。

  ---
  
  通过上面的分析我们可以看到，每次进行函数调用我们都需要为被调用的函数分配栈空间，用以保存调用参数和局部变量。为了实现协程，我们必须要实现前面说的“暂停”和“恢复”特性。有栈协程为了实现这两个特性做法是想办法在暂停后保存这些参数、变量以及相关的寄存器，在需要恢复调用前再将保存的信息恢复。在`Linux`中可以调用`ucontext`族函数`getcontext`、`setcontext`等实现<sup>[[12]](#ref_12)</sup>；`Boost`库也有类似的实现`Boost.Context`<sup>[[13]](#ref_13)</sup>；甚至可以自己通过汇编实现，例如微信的`libco`<sup>[[14]](#ref_14)</sup>。


  无栈协程不需要此过程。 在《使用 C 语言实现协程》<sup>[[15]](#ref_15)</sup>文章作者详细介绍了一种利用达夫设备<sup>[[16]](#ref_16)</sup>原理使用C语言实现协程的方法。由于原理比较简单在此粗略的介绍下其核心代码：
  ```c++
  int function(void) {
    static int i, state = 0;
    switch (state) {
        case 0:               // 函数开始执行
        for (i = 0; i < 10; i++) {
            state = 1;        // 我们会回到 "case 1" 的地方
            return i;
            case 1:;          // 从上一次返回之后的地方开始执行
        }
    }
    return i;
  }
  ```
  <center><div style="color:orange; display: inline-block; color: #999; padding: 2px;">代码6</div></center></br>
  这段奇怪的代码之所以能够正确执行是因为C语言标准规定：“在switch控制语句内，条件标号（case）可以出现在任意子语句之前，充作其前缀”<sup>[[16]](#ref_16)</sup>。现在分析下代码：
    
    第1次调用`function()`时, state值为0，所以执行case 0，然后执行for循环体，i赋0， state赋1(相当于保存状态，暂停执行)，函数返回i(0)。</br>
    第2次调用`function()`时, state值为1，执行case 1（相当于恢复状态，继续执行），执行for循环后面的语句，i++， state赋1，函数返回i(1)。</br>
    第3次调用`function()`时, state值为1，执行case 1，执行for循环后面的语句，i++， state赋1，函数返回i(2)。</br>
    ...
    
    第10次调用`function()`时, state值为1，执行case 1，执行for循环后面的语句，i++， state赋1，函数返回i(9)。</br>
    第11次调用`function()`时, state值为1，执行case 1，因为i>=10所以跳出循环，函数返回i(9)。</br>

  由此可见，通过一个状态量，便可改变每次函数的执行流程。看到这里也许你会有疑问，为什么要用这么复杂的语法，if-else不也可以实现吗？其实这样写主要方便利用宏进行代码封装，具体的做法可见参考文献<sup>[[15]](#ref_15)</sup>。其实在C++协程标准的实现里面也有类似的做法，只不过是编译器通过汇编指令直接跳转实现的，具体的分析后面会介绍。

- 它们通过返回到调用方暂停执行。这里说的是协程的非对称性，协程按照调度方式可以分为对称协程和非对称协程<sup>[[17]](#ref_17)</sup>。

  - 对称协程 Symmetric Coroutine：任何一个协程都是相互独立且平等的，调度权可以在任意协程之间转移。
  - 非对称协程 Asymmetric Coroutine：协程出让调度权的目标只能是它的调用者，即协程之间存在调用和被调用关系。

  直白的说就是， 对称协程只提供了一种类似于goto的操作将控制权直接传递给其它指定的协程，而非对称协程需要前面所说的“暂停”和“恢复”两种操作。对于对称协程调用者和被调用者之间的关系是平等的，他们甚至可以互相调用，可以做到A->B->C->A（A,B,C为不同的协程函数）这样的链式调用。非对称协程具有明确的层级关系被调用者必须先返回到调用者再调用其他协程，如A->B->A->C->A。粗看好像对称协程更容易使用，因为其具有更大的自由度，但是它和goto遇到的问题是一样的，在一个复杂的系统里过多的自由跳转会破坏代码的结构，让程序逻辑不容易得到把控。例外对称协程只是非对称协程的一个特例，我们可以通过添加一个中立的第三方调度中心的方式将非对称协程转换成对称协程（只需要在所有协程“暂停”时将控制权转交给调度中心统一调度即可）。

- 并且从栈分离存储恢复所要求的数据。这里所说是相对于普通的函数调用，如图1，图2所示，我们在调用函数时总是会先将调用参数和临时变量以及一些相关寄存器信息压入堆栈，在后面的指令需要时再从堆栈中取出，最后函数执行完后清理掉。但是协程具有额外的“暂停”和“恢复”操作，如果还是使用堆栈上的信息，根据堆栈的使用规则，协程“恢复”后堆栈已经遭到破坏，因为“暂停”后为了使调用者能够正常工作，我们必须保证此时的堆栈状态能够恢复到调用者的堆栈状态，“恢复”后由于协程的堆栈信息已经丢失，我们无法再使用协程的堆栈。所以我们需要额外的手段将这些信息存储在其他地方（一般是在堆上）。值得注意的是这里的用词是“分离”而不是复制，具体实现后面做详细的分析。

- 这允许编写异步执行的顺序代码（例如不使用显式的回调来处理非阻塞 I/O）。 参考前面代码4的例子。

- 还支持对惰性计算的无限序列上的算法及其他用途。
  - 无限序列（infinite sequence）是指一列或一串离散的数字能够与另一组正整数$\\{0, 1, 2, 3, ...\\}$相比而形成一定的联系和概念。例如无限序列 $N=(0, 1, 2, 3, ...)$ 和 $S＝(1, \frac{1}{2}, \frac{1}{4}, \frac{1}{8}, ..., \frac{1}{2^n}, ...)$, 当然斐波纳契数列$F＝(1, 1, 2, 3, 5, 8, ..., n_{i-1}, n_{i}, n_{i-1} + n_{i}, ...)$也是一种无限序列。
  - 惰性计算（又称延迟求值）在科学计算和函数式编程中经常用到的一种优化手段。引用`Wiki`中给的例子简单说明下， 下面是部分代码<sup>[[19]](#ref_19)</sup>:
    ```cpp
    matrix A,B;
    A[3][4]=101; B[3][4]=102;
    matrix_add R=A+B;
    int result = R[3][4];
    ```
    上面的代码是计算两个矩阵（A, B）相加后得到新矩阵(R)某个索引的值。通常我们在执行完`R=A+B`时便会算出R中的所有元素，因为A和B均有$12=3\times4$个元素，所以为了得到R需要经过12次加法运算，但是我们只需要`R[3][4]`这一个元素的值，所以我们浪费了11次运算。惰性计算的方法是执行到`R=A+B`时，我们并不去计算，而只是存储计算所需要的数据，计算被延迟到了取`R[3][4]`值时才开始。完整的代码请参考`Wiki`<sup>[[19]](#ref_19)</sup>。

  - 我们通过`cppcoro`计算斐波纳契的例子来说明下C++协程是怎么实现惰性计算的。
    
    ```cpp
    cppcoro::generator<const std::uint64_t> fibonacci()
    {
      std::uint64_t a = 0, b = 1;
      while (true)
      {
        co_yield b;                     // 2
        auto tmp = a;
        a = b;
        b += tmp;
      }
    }

    void usage()
    {
      for (auto i : fibonacci())         // 1
      {
        if (i > 1'000'000) break;
        std::cout << i << std::endl;
      }
    }
    ```

    在代码注释1处，我们通过循环不停调用fibonacci， 第一次调用fibonacci时从函数入口开始初始化a, b，然后进入循环，执行 `co_yield b; `这是会从fibonacci函数回到usage，执行for循环体中代码，再次调用fibonacci时实际上执行的是fibonacci协程的的resume方法，所以再次进入fibonacci函数时并不是从入口重新执行，而是恢复到co_yield后面的语句`auto tmp = a;`继续执行算出数列的下个值b，然后循环再次`co_yield`回到usage，如此循环往复，最终得到数列中第1'000'000个值。 </br>
    可以看到第一次调用时我们并没有把fibonacci数列中的前1'000'000个值全部计算出来，而是把计算推迟到了后面的调用。如果有足够的时间和算力我们可以计算任意数列的值，从算法上并不限制数列的长度，可以无限的计算下去， 所以说是“无限序列”。

- 若函数的定义做下列任何内容之一，则它是协程。 简单的说就是如果一个函数中出现co_await、co_yield、co_return 三个关键字，你就可以认为它是一个协程函数。


 ## 原理剖析 
  `C++`语言的难学已经是恶名远播，提起C++时许多程序员（甚至是学C语言的程序员）第一反应就是“这是个令人胆颤、难缠、恶心的家伙”。其实换个角度`C`与`C++`的关系非常类似于`JavaScript`和`Typescript`(如果你用过应该很好理解)，C++作为C的超集经过编译器的转换绝大部分的代码是可以转换成C，因为有了编译器的帮助C++可以实现许多高级的封装，每次编译时编译器都会根据需要向用户编写的代码里“偷偷”加料。例如构造析构函数的自动创建、虚函数表、this指针传递等等，对于这方面的知识建议大家看看《深入了解c++内核对象模型》这本书。因为编译器的这些小动作对用户是无感知的，这就造成了用户对C++一些行为表现的困惑，也加大了大家学习C++的难度。而协程特性是我目前见到C++让编译器加料最多的一次，理所当然理解的难度指数也是陡增。现在就让我们细细的分析下编译器给我们加的这些料的配方吧！



### 协程函数展开
---
  让我们看个最简单的例子，看看`main`函数调用`remote_query_all`时发生了什么。

  ```cpp
  #include <experimental/coroutine>

  return_ignore test_coroutine() {
    co_await std::experimental::suspend_always{};
  }

  int main() {
    test_coroutine();

    return 0;
  }
  ```

  其中`return_ignore`的定义如下：
  ```cpp
  struct return_ignore {
    struct promise_type {
      return_ignore get_return_object() {
        return return_ignore{};
      }

      auto initial_suspend() {
        return std::experimental::suspend_always{};
      }

      auto final_suspend() {
        return std::experimental::suspend_never{};
      }

      void unhandled_exception() {
      }

      void return_void() {}
    };
  };
  ```


未完待续...



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

[9]  <a id="ref_9" href="https://zh.wikipedia.org/zh-cn/%E5%8D%8F%E7%A8%8B" >协程
</a>`[EB/OL].`https://zh.wikipedia.org/zh-cn/%E5%8D%8F%E7%A8%8B

[10]  <a id="ref_10" href="https://github.com/Tencent/libco" >Tencent/libco</a>`[EB/OL].`https://github.com/Tencent/libco

[11]  <a id="ref_11" href="http://www.golangpatterns.info/concurrency/coroutines" >Go Language Patterns/Coroutines</a>`[EB/OL].`http://www.golangpatterns.info/concurrency/coroutines

[12]  <a id="ref_12" href="https://linux.die.net/man/2/setcontext" >setcontext(2) - Linux man page</a>`[EB/OL].`https://linux.die.net/man/2/setcontext

[13]  <a id="ref_13" href="https://www.boost.org/doc/libs/1_74_0/libs/context/doc/html/context/overview.html" >Boost.Context</a>`[EB/OL].`https://www.boost.org/doc/libs/1_74_0/libs/context/doc/html/context/overview.html

[14]  <a id="ref_14" href="https://github.com/Tencent/libco" >Libco</a>`[EB/OL].`https://github.com/Tencent/libco

[15]  <a id="ref_15" href="https://mthli.xyz/coroutines-in-c/" >使用 C 语言实现协程</a>`[EB/OL].`https://mthli.xyz/coroutines-in-c/

[16]  <a id="ref_16" href="https://en.wikipedia.org/wiki/Duff%27s_device" >达夫设备</a>`[EB/OL].`https://en.wikipedia.org/wiki/Duff%27s_device

[17]  <a id="ref_17" href="https://www.bennyhuo.com/2019/12/01/coroutine-implementations/" >破解 Kotlin 协程 番外篇(2) - 协程的几类常见的实现</a>`[EB/OL].`https://www.bennyhuo.com/2019/12/01/coroutine-implementations/

[18]  <a id="ref_18" href="https://searchcio.techtarget.com.cn/whatis/8-28809/" >infinite sequence：无限序列</a>`[EB/OL].`https://searchcio.techtarget.com.cn/whatis/8-28809/

[19]  <a id="ref_19" href="https://zh.wikipedia.org/wiki/%E6%83%B0%E6%80%A7%E6%B1%82%E5%80%BC" >惰性求值</a>`[EB/OL].`https://zh.wikipedia.org/wiki/%E6%83%B0%E6%80%A7%E6%B1%82%E5%80%BC

[20]  <a id="ref_20" href="https://github.com/lewissbaker/cppcoro" >CppCoro - A coroutine library for C++</a>`[EB/OL].`https://github.com/lewissbaker/cppcoro

## 未整理部分

天涯明月刀-无栈协程的应用 https://gameinstitute.qq.com/community/detail/107515

有栈协程与无栈协程 https://mthli.xyz/stackful-stackless/

luncliff/coroutineExploring MSVC Coroutine 
https://luncliff.github.io/coroutine/articles/exploring-msvc-coroutine/
https://luncliff.github.io/posts/Exploring-MSVC-Coroutine.html
https://luncliff.github.io/coroutine/ppt/%5BEng%5DExploringTheCppCoroutine.pdf

coroutines https://github.com/topics/coroutines?l=c%2B%2B

汇编 https://cs.lmu.edu/~ray/notes/nasmtutorial/
汇编编译 https://godbolt.org/

https://luncliff.github.io/coroutine/ppt/%5BEng%5DExploringTheCppCoroutine.pdf

N3858.pdf https://isocpp.org/files/papers/N3858.pdf

达夫设备 https://mthli.xyz/coroutines-in-c/

其实上面说的是有一些问题，因为这个执行权的切换实际上是（调用者–被调用者）之间的切换，对称就是它们之间都是平等的，就是假如A协程执行了B，C协程，那么B协程可以切换回A，也可以切换回C。而非对称只能是B切换回A，A切换回C，C再切换回A，以此类推。
https://blog.csdn.net/weixin_43705457/article/details/106924435

callback hell https://medium.com/@quyetvv/async-flow-from-callback-hell-to-promise-to-async-await-2da3ecfff997

 [The reasons why 64-bit programs require more stack memory](https://software.intel.com/content/www/us/en/develop/blogs/the-reasons-why-64-bit-programs-require-more-stack-memory.html)   https://software.intel.com/content/www/us/en/develop/blogs/the-reasons-why-64-bit-programs-require-more-stack-memory.html

 C++ Coroutines: Understanding Symmetric Transfer https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer 

C++网络编程之asio(五)——在asio中使用协程 https://zhuanlan.zhihu.com/p/58784652

# 