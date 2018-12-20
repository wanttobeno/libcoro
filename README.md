
C++11实现的协程库，支持Win，Linux，Mac。

Fork Form https://github.com/mfichman/coro/commit/2d597a7ebe08bc28d91b98c942be17eb224b8853

协同程序(coroutine)简称协程

##### 协同程序

参考Lua中协同程序的介绍

http://www.runoob.com/lua/lua-coroutine.html

---


##### Coroutine 协程类

协程类,用于封装函数。一个协程只能包含一个函数，在执行期间的任何时刻可暂停和恢复。每个协程分配CORO_STACK_SIZE(1M)的内存大小。

```c++
class Coroutine : public std::enable_shared_from_this<Coroutine> {

Coroutine(F func)；  // 构造函数，参数是一个函数，作为绑定函数

void start() throw();  // 执行绑定函数
内部 func_();  // 执行绑定的函数
	 event_->notifyAll();

void exit(); //当协程完成执行时，此函数运行。
内部event_->notifyAll();
	main()->swap();

void swap(); // 将运行权给其它协程，当前正在执行的协程会挂起。

void yield();  // 挂起coroutine，将coroutine设置为挂起状态。RUNNING状态切换为RUNNABLE状态，下次循环会继续运行


void block(); // 阻止当前协同程序，直到发生某些I/O事件。 协程会在明确安排之前不得重新安排。RUNNING状态切换为BLOCKED状态
内部hub()->blocked_++;

void unblock(); // 解除阻止
内部hub()->blocked_--;

void wait(); // 阻止当前协同程序，直到发生某些事件。 协程不会被重新安排，直到明确安排。BLOCKED状态切换为RUNNABLE状态
内部hub()->waiting_++;

void notify(); // 在事件发生时取消阻止协同程序。
内部hub()->waiting_--;

// 部分成员变量
Ptr<Event> event_;  // 信号
}
```

##### Coroutine 状态切换

这个状态的过程比较的重要！

1、NEW --> RUNNING --> EXITED

2、RUNNING --> BLOCKED --> RUNNABLE --> RUNNING --> EXITED

3、WAITING --> RUNNABLE --> RUNNING --> EXITED

```c++
Coroutine::Coroutine(); // 初始化状态为RUNNING状态

Coroutine::~Coroutine(): // 切换为DELETED状态

void Coroutine::wait(); // RUNNING切换为WAITING状态

void Coroutine::notify(); // WAITING切换为RUNNABLE状态

void Coroutine::swap(); // 当前RUNNABLE切换为RUNNING；NEW切换为RUNNING;BLOCKED切换为RUNNING。最重要的函数，控制协程的中断，切换。

void Coroutine::exit(); // RUNNING切换为EXITED状态

void Coroutine::yield(); // RUNNING切换为RUNNABLE状态

void Coroutine::block(); // RUNNING切换为BLOCKED状态

void Coroutine::unblock(); // BLOCKED切换为RUNNABLE状态


```

##### Coroutine 内存分配

每个协程分配CORO_STACK_SIZE(1M)的内存大小,保存到成员变量
Stack stack_;,所分配的内存会在Coroutine析构函数执行后释放。

```c++
if defined(_WIN32)
struct StackFrame {
    void* fs8;		// 栈顶
    void* fs4;		// 栈低
    void* fs0;		// Root-level SEH handler
    void* rdi;
    void* rsi;
    void* rdx;
    void* rcx;
    void* rbx;
    void* rax;
    void* rbp;
    void* returnAddr; // coroStart() stack frame here
};
```
stackPointer_ // 保存初始化的信息地址，未被调用。

Coroutine析构的时候释放内存。


##### Event 协程同步事件

协程的调用过程是单线程的，Event的控制是通过改变协程的状态来控制的。

```c++
void notifyAll(); // 通知所有协程运行，添加到std::vector<EventRecord>容器中
void wait();  // 等待协程信号。实际是将协程的RUNNING切换为WAITING状态并挂起，调用其他的协程。
```

##### Hub 管理协程容器

Hub类管理所有coroutines，events和I/O。
等待的事件（通道,I/O等）发出完成信号，在执行调用。

被定义为一个静态变量，通过coro::run()调用。

```c++
void quiesce(); // 遍历runnable_协程容器，更新协程的状态(void Coroutine::swap())，将RUNNABLE状态的协程保存到runnable_容器。

void poll(); // 轮询I/O事件。

void Hub::run(); // 循环运行，每次取一个协程
```

##### 常用API

```C++
// 添加函数到协程容器
coro::start(baz);

// 挂起，RUNNING状态切换为RUNNABLE状态
coro::yield();

// 函数中调用，用于定时中断，中断时间到，继续执行
coro::sleep(coro::Time::millisec(1000));

// 执行协程容器
coro::run(); 
```


##### Demo1 - Basic 基础例子

```c++
#include "coro/Common.hpp"
#include "coro/Coroutine.hpp"
#include "coro/Hub.hpp"

void bar() {
    for (auto i = 0; i < 2; ++i) {
        coro::sleep(coro::Time::millisec(1000));
        std::cout << "barrrrrr" << std::endl;
    }
}

void baz() {
    for (auto i = 0; i < 20; ++i) {
        coro::sleep(coro::Time::millisec(100));
        std::cout << "baz" << std::endl;
    }
}

int main() {
    auto cbaz = coro::start(baz);
    auto cbar = coro::start(bar);
    coro::run();
    return 0;
}
```

##### Demo2 - Event 带信号的例子

```c++
#include <coro/Common.hpp>
#include <coro/coro.hpp>

int main() {
    auto event = coro::Event();
    auto trigger = false;

    auto notifier = coro::start([&]() {
        printf("notified\n");
        trigger = true;
        event.notifyAll();	// 切换所有WAITING状态的协程为RUNNABLE状态
    });

    auto waiter = coro::start([&]() {
        printf("waiting\n");
        event.wait([&]() { return trigger; });  // 等待信号
        printf("done\n");
    });

    coro::run();

    return 0;
}
```

##### Demo3 - Join 嵌套例子

```c++
#include <coro/Common.hpp>
#include <coro/coro.hpp>

int main() {
    auto counter = 0;
    auto one = coro::start([&]{
        coro::yield();  // 挂起，RUNNING切换为RUNNABLE状态。恢复的时候继续运行下去
        assert(counter==0);
        counter++;
		std::cout << "one func" << counter << std::endl;
    });
    auto two = coro::start([&]{
        one->join();  // two状态有RUNNING改为WAITING；one状态由RUNNABLE改为RUNNING。
        assert(counter==1);
		std::cout <<"two func" << counter << std::endl;
    });

    coro::run();  // 
    return 0;
}
```

##### Demo4 - Selector  信号和协程绑定

使用这个类，开发者可以决定是否执行协程函数。
Selector局部变量析构函数中会将协程的状态由RUNNING切换为WAITING。Selector中可以保持多个协程对象。

```c++

#include <coro/Common.hpp>
#include <coro/coro.hpp>

using namespace coro;

Ptr<Event> e1(new Event);
Ptr<Event> e2(new Event);
Ptr<Event> e3(new Event);

void publisher() {
    e1->notifyAll();	// 通知执行
    e2->notifyAll();	// 通知执行
}

void consumer() {
    int count = 2;
    while (count > 0) {
        coro::Selector()  // 绑定信号和协程对象
            .on(e1, [&]() { count--; std::cout << "e1" << std::endl; })
            .on(e2, [&]() { count--; std::cout << "e2" << std::endl; })
            .on(e3, [&]() { std::cout << "e3" << std::endl; });
    }
}

int main() {
    auto b = coro::start(consumer);
    auto a = coro::start(publisher);

    coro::run();

    return 0;
}
```

