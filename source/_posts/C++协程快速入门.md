---
title: C++协程快速入门
date: 2026/1/30 17:03:00
categories: 
---
## 简介
协程是一种可以暂停执行并在之后恢复的函数。协程是无栈的，当协程被暂停时，协程的整体状态会存储在一个与栈分离的对象中，这样它就可以在完全不同的上下文（不同的调用栈、另一个线程等）中被恢复。

只要在函数中使用以下关键字之一，就会隐式定义一个协程：
- `co_await` 表达式—用于暂停执行直到恢复
- `co_yield` 表达式—用于暂停执行并返回一个值
- `co_return` 语句—用于完成执行并返回一个值
## 一个协程示例
如下是一个使用协程的示例程序，先只看 `main` 函数和 `counter` 函数，暂且忽略 `Result` 类型。
```C++
#include <coroutine>
#include <iostream>

struct Result {
    struct promise_type {
        Result get_return_object() noexcept {
            return {
                .h_ = std::coroutine_handle<promise_type>::from_promise(*this)
            };
        }

        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_void() noexcept {
        }

        void unhandled_exception() noexcept {
        }
    };

    std::coroutine_handle<promise_type> h_;
};

Result counter() {
    for (int i = 0;; ++i) {
        printf("%d\n", i);
        co_await std::suspend_always{};
    }
}

int main() {
    auto coro = counter();
    for (int i = 0; i < 3; ++i) {
        coro.h_.resume();
    }
}
```
运行结果：
```
0
1
2
3
```
这里 `counter` 是一个无限循环的协程函数。新建协程的时候，协程开始执行直到 `co_await` 表达式，`co_await` 会挂起当前协程，返回 `main` 函数。

暂停协程的具体行为由 `co_await` 后面的 `awaitable` 对象定义。它使程序员能够控制暂停的具体行为。

目前，我们使用 `std::suspend_always` 的默认构造对象，它接受暂停并将控制权交回给调用者。

控制权返回到 `main` 函数后，`main` 函数中调用 ***协程句柄（coroutine handle）*** 的 `resume()` 函数会恢复协程，让协程在挂起点后继续运行。即使控制流在 `counter` 函数与调用它的 `main` 函数之间反复切换，变量 `i` 仍能不受影响地不断累加。

***协程句柄（coroutine handle）*** 是一个协程的标识，用于恢复协程执行或销毁协程。

之前暂时被忽略 `Result` 为协程的返回类型，C++ 要求协程函数的返回类型具有嵌套类型 `Result::promise_type`，也称之为 ***promise*** 类型（见[`Promise 对象`](#Promise-对象)章节）。

当协程开始启动时，会分配 ***协程状态（coroutine state）*** 对象，保存协程函数的参数，临时变量，挂起点和 ***promise*** 对象。这样，当协程恢复执行时，就可以在暂停运行的位置后继续运行。
## Promise 对象
***Promise*** 对象通过实现一些接口方法，来定义和控制协程本身的行为。注意，***promise*** 对象与 `std::promise` 没有任何关联。

Promise 对象需要至少实现下面的五个函数：
- `get_return_object()`：获得协程对象。
- `initial_suspend()`：允许进行额外的初始准备，并定义协程是应该立即启动还是延迟启动：
	- 返回 `std::suspend_never{}` 意味着立即启动。
	- 返回 `std::suspend_always{}` 意味着延迟启动。协程会立即挂起，不执行任何语句。它会在恢复时开始启动。
- `final_suspend()`：和上面类似，只不过是在协程结束时，定义了协程是否应该挂起，**该函数必须是 `noexcept` 的**。
	- 返回 `std::suspend_never{}` 自动释放相关资源。
	- 返回 `std::suspend_always{}` 协程挂起，需要用户释放相关资源。
- `unhandled_exception()`：当协程因异常而结束时，调用该函数处理异常。
- `return_void()/return_value()`：协程执行到末尾（或遇到 `co_return` 语句）时会执行的函数。
## co_yield
如果 `p` 是promise对象，那么表达式 `co_yield e` 等价于 `co_await p.yield_value(e)`。

一个简单使用 `co_yield` 的例子：
```c++
#include <coroutine>  
#include <iostream>  
  
struct Result {  
    struct promise_type {  
        Result get_return_object() noexcept {  
            return {  
                .h_ = std::coroutine_handle<promise_type>::from_promise(*this)  
            };  
        }  
  
        std::suspend_never initial_suspend() noexcept { return {}; }  
        std::suspend_always final_suspend() noexcept { return {}; }  
  
        void return_void() noexcept {  
        }  
  
        void unhandled_exception() noexcept {  
        }  
  
        std::suspend_always yield_value(int value) {  
            value_ = value;  
            return {};  
        }  
  
        int value_;  
    };  
  
    std::coroutine_handle<promise_type> h_;  
  
};  
  
Result counter() {  
    for (int i = 0;; ++i) {  
        co_yield i;  
    }  
}  
  
int main() {  
    auto coro = counter();  
    for (int i = 0; i < 3; ++i) {  
        std::cout << coro.h_.promise().value_ << std::endl;  
        coro.h_.resume();  
    }  
}
```
输出为：
```
0
1
2
```
## co_return
`co_return` 指示一个协程的结束点。协程有三种方法示意结束：
1. `co_return e`：协程会执行 `p.return_value(e)` 语句。
2. `co_return`：协程会执行 `p.return_void()` 语句。
3. 协程自然的运行到函数结尾：同方法 2。

由之前章节可知，协程结束时不代表协程一定会被销毁。

一个使用 `co_return` 的示例：
```c++
#include <coroutine>  
#include <iostream>  
  
struct Result {  
    struct promise_type {  
        ~promise_type() {  
            std::cout << "promise_type destroyed" << std::endl;  
        }  
  
        Result get_return_object() noexcept {  
            return {  
                .h_ = std::coroutine_handle<promise_type>::from_promise(*this)  
            };  
        }  
  
        std::suspend_never initial_suspend() noexcept { return {}; }  
        std::suspend_always final_suspend() noexcept { return {}; }  
  
        void return_value(int value) noexcept {  
            value_ = value;  
        }  
  
        void unhandled_exception() noexcept {  
        }  
  
        int value_;  
    };  
  
    std::coroutine_handle<promise_type> h_;  
  
};  
  
Result counter() {  
    co_return 1024;  
}  
  
int main() {  
    auto coro = counter();  
    std::cout << coro.h_.promise().value_ << std::endl;  
    coro.h_.destroy();  
}
```
输出为：
```
1024
promise_type destroyed
```
## Awaitable对象
执行“**暂停**”这一操作的运算符是 `co_await`，它需要一个 `awaitable` 对象，该类型主要需要实现下面的函数：
- `await_ready()`：这个函数在协程即将挂起之前被调用，如果返回 `false`，那么 `co_await awaitable` 就会立即暂停该协程。这个函数通常只返回 `false`，但如果挂起没有意义时可能返回 `true`。
- `await_suspend(std::coroutine_handle<> handle)`：如果协程被挂起，该函数会被调用，其中 `handle` 参数就是调用等待体的协程，函数的返回值有以下可能：
	- `void`：同 `true`。
	- `bool`：返回 `true`，协程暂停，将控制返还给调用者，返回 `false` 立即恢复该协程。
	- 返回某个 ***协程句柄（coroutine handle）***，立即恢复对应协程的运行。
- `await_resume()`：协程挂起后恢复时，调用的接口。返回值作为 `co_wait` 操作的返回值。

标准库定义了两个简单的可等待对象，之前已经被多次提到：
- `std::suspend_never` 类，不挂起的的特化等待体类型。
- `std::suspend_always` 类，挂起的特化等待体类型。
