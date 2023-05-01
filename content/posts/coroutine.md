+++
title = "协程的 C 语言实现"
date = "2023-04-30T16:42:25+08:00"
author = "Civitasv"
authorTwitter = "" #do not include @
cover = ""
tags = ["coroutine", "C"]
keywords = ["协程", "C 语言实现"]
description = "本文介绍如何在 C 语言中实现协程"
showFullContent = false
readingTime = true
hideComments = false
color = "blue" #color from the theme settings
+++

## 调研

### 1. From http://wiki.c2.com/?CoRoutine

Coroutines are very important as one of the very few examples of a form of concurrency that is useful, and yet constrained enough to completely avoid the typical difficulties (race conditions, deadlock, etc).

Synchronization is built in to the paradigm. It therefore cannot in general replace more general unconstrained forms of concurrency, but for some things it appears to be the ideal solution.

For example:

```c
int coroutine Generate123() {
    yield 1;  /* Execution begins here upon first call to Generate123 */
    yield 2;  /* execution continues here upon "resume Generate123" */
    yield 3;  /* execution continues here upon second "resume Generate123" */
}

main() {
    printf("%d\n", Generate123());	/* prints "1" */
    printf("%d\n", resume Generate123()); /* prints "2" */
    printf("%d\n", resume Generate123()); /* prints "3" */
}
```

### 2. 协程在 State Machine 的应用

{{< youtube MuCpdoIEpgA >}}

### 3. From https://www.csl.mtu.edu/cs4411.ck/www/NOTES/non-local-goto/coroutine.html

单线程并发执行 `Ping` 和 `Pong`:

{{< code language="c" title="简单的协程实现（无上下文切换）" id="1" expand="Show" collapse="Hide" isCollapsed="true" >}}
/_ ---------------------------------------------------------------- _/
/_ PROGRAM pingpong : _/
/_ This program uses setjmp() and longjmp() to implement an _/
/_ example of coroutine. _/
/_ ---------------------------------------------------------------- _/

#include <stdio.h>
#include <setjmp.h>
#include <stdlib.h>

int max_iteration; /_ the max # of iterations _/
int iter; /_ global iteration counter _/

jmp_buf Main; /_ jump back to main() _/
jmp_buf PointA; /_ jump buffer in Ping() _/
jmp_buf PointB; /_ jump buffer in Pong() _/

/_ ---------------------------------------------------------------- _/
/_ Function Prototypes _/
/_ ---------------------------------------------------------------- _/

void Ping(void);
void Pong(void);

/_ ---------------------------------------------------------------- _/
/_ The main program starts here _/
/_ ---------------------------------------------------------------- _/

int main(int argc, char* argv[])
{
    if (argc != 2) { /* check # of arguments _/
        printf("Use %s max-#-of-lines\n", argv[0]);
        exit(1);
    }
    max_iteration = abs(atoi(argv[1]));/_ get max # of iterations _/
    iter = 1; /_ initial iteration count _/
    if (setjmp(Main) == 0) /_ set a return mark _/
        Ping(); /_ initialize Ping() _/
    if (setjmp(Main) == 0) /_ set a return mark _/
        Pong(); /_ initialize Pong() _/
    longjmp(PointA, 1); /_ ok, jump to Ping() \*/
}

/_ ---------------------------------------------------------------- _/
/_ FUNCTION Ping : _/
/_ This function marks a return point when it is initialized. _/
/_ Then, it starts a loop and jump back and forth between itself _/
/_ and function Pong() using jump buffers. _/
/_ ---------------------------------------------------------------- _/

void Ping(void)
{
    if (setjmp(PointA) == 0) /_ set a return mark _/
        longjmp(Main, 1); /_ jump back to main _/
    while (1) { /_ main will jump to here _/
	    printf("%3d : Ping-", iter); /_ display Ping _/
        if (setjmp(PointA) == 0) /_ set a return mark _/
            longjmp(PointB, 1); /_ jump to Pong() _/
    }
}

/_ ---------------------------------------------------------------- _/
/_ FUNCTION Pong : _/
/_ This function marks a return point when it is initialized. _/
/_ Then, it starts a loop and jump back and forth between itself _/
/_ and function Ping() using jump buffers. _/
/_ ---------------------------------------------------------------- _/

void Pong(void)
{
    if (setjmp(PointB) == 0) /_ set a return mark _/
        longjmp(Main, 1); /_ jump back to main _/
    while (1) { /_ main will jump to here _/
        printf("Pong\n"); /_ display Pong _/
        iter++; /_ increase iteration count _/
        if (iter > max_iteration) /_ should I stop? _/
            exit(0); /_ yes, then exit _/
        if (setjmp(PointB) == 0) /_ no, set a return mark _/
            longjmp(PointA, 1); /_ then jump to Ping() _/
    }
}
{{< /code >}}

## 什么是协程

协程常被称为轻量级线程，其不同于线程和进程，协程之间的切换不会有内核态与用户态之间的切换，切换开销较小。当任务为 I/O 密集型时，采取多线程技术，线程之间的切换会很频繁，这时候应当使用协程。

- 对于线程，操作系统负责具体的调度，也就是说，线程的暂停与恢复是操作系统完成的。
- 但对于协程，程序员负责具体的调度，也就是说，协程的暂停与恢复是程序员完成的。

协程在线程内创建，线程内多个协程的运行是串行的，当线程内的某一个协程运行时，其它协程必须挂起，因此，对于计算密集型任务，协程并没有任何意义，而对于 I/O 密集型任务，配合异步编程，协程在 I/O 等待时就去执行其它任务，I/O 操作结束后再自动回调，就可以大大节省资源并提高性能。

> 详细解释：协程完全存在于用户态，操作系统是不知道协程的存在的，其基本调度单位是线程，所以如果协程调用阻塞 I/O 时，操作系统会认为线程阻塞，当前的协程和其它绑定在该线程之上的协程都会陷入阻塞而得不到调度，所以协程必须与异步 I/O 结合起来使用。

## 为什么需要协程

1. 相比于线程，协程的调度不需要操作系统的干预，因此开销很低，可以提供高层次的并发
2. 程序员可以完全控制协程的暂停与恢复，因此不需要对公用数据加锁

## Implementation

### 上下文切换

#### ucontext_t

```c
typedef struct {
    ucontext_t *uc_link;
    stack_t     uc_stack;
    mcontext_t  uc_mcontext;
    sigset_t    uc_sigmask;
    ...
} ucontext_t;
```

其中:

- `uc_link` 指向下一个启动的 `context`
- `uc_stack` 存储当前 `context` 使用的栈信息
- `uc_mcontext` 存储运行状态，包括寄存器、CPU 状态、指令指针、栈指针等信息
- `uc_sigmak` 存储当前 `context` 中阻塞的一组信号

然后，我们使用以下四个方法完成协程的上下文切换。

#### setcontext(const ucontext_t \*ucp)

将控制权转移给 ucp，使程序从存储在 ucp 中的位置继续，相当于 longjmp。

#### getcontext(ucontext_t \*ucp)

将当前上下文存储到 ucp 中，相当于 setjmp。

#### makecontext(ucontext_t \*ucp, void(\*func)(), int argc, ...)

创建上下文。

#### swapcontext(ucontext_t *oucp, ucontext_t *ucp)

将当前运行状态存储到 oucp 中，然后将控制权转移到 ucp 中。

#### Example

```c
void context_switching()
{
	ucontext_t ctx = {0};

	getcontext(&ctx); // store context
	printf("Context switching example\n");
	sleep(1);
	setcontext(&ctx); // jump to location of ctx
}
```

该例会无限循环输出 `Context switching example`。

```c
void control_flow()
{
	uint32_t var = 0;
	ucontext_t ctx = { 0 }, back = { 0 };

	getcontext(&ctx); // 1. store current context in ctx

	ctx.uc_stack.ss_sp = calloc(1, MINSIGSTKSZ);
	ctx.uc_stack.ss_size = MINSIGSTKSZ;
	ctx.uc_stack.ss_flags = 0;

	ctx.uc_link = &back;

	makecontext(&ctx, (void (*)())another, 2, &var, 100);

	// 2. store current context in back and move control to ctx
	swapcontext(&back, &ctx);

	printf("var = %d\n", var); // 4. print
}
```

1. `getcontext(&ctx)` 会将当前上下文存储到 ctx 中
2. 为 `ctx.uc_stack` 赋值
3. 为 `ctx.uc_link` 赋值为 `back`
4. 使用 `makecontext` 创建一个心得上下文，并将其与 `another` 函数绑定
5. 使用 `swapcontext` 将当前上下文存储到 `back`，然后将控制权转移到 `ctx`
6. `another` 函数执行完毕后，会启动 `back` 上下文，打印 `var`

### 实现协程

#### 接口设计

```c
#ifndef CCOROUTINE_H_
#define CCOROUTINE_H_

#include <ucontext.h>
#include <ucontext.h>
#include <stdbool.h>

typedef struct ccoroutine ccoroutine;
typedef void *(*ccoroutine_function)(ccoroutine *coro, void *userdata);

// coroutine handler
struct ccoroutine {
	ccoroutine_function 	function;		// 该协程要执行的函数
	ucontext_t 		suspend_context;	// 挂起的上下文
	ucontext_t 		resume_context;		// 当前执行的上下文
	void 			*yield_value;		// 生成值
	bool			is_finished;		// 协程是否已结束执行
};

// apis
ccoroutine	*ccoroutine_create(ccoroutine_function function, void* userdata);
void		*ccoroutine_resume(ccoroutine *coro);
void 		ccoroutine_yield(ccoroutine* coro, void *value);
void		ccoroutine_destroy(ccoroutine* coro);
bool		ccoroutine_finished(ccoroutine* coro);

#endif // CCOROUTINE_H_
```

#### 实现

```c
#include "ccoroutine.h"
#include <stdlib.h>
#include <signal.h>

static void ccoroutine_entry_point(ccoroutine *coro, void* userdata)
{
	void *return_val		= coro->function(coro, userdata);
	coro->is_finished		= true;
	ccoroutine_yield(coro, return_val);
}

ccoroutine *ccoroutine_create(ccoroutine_function function, void* userdata)
{
	ccoroutine *coro = calloc(1, sizeof(*coro));

	coro->is_finished			= false;
	coro->function				= function;
	coro->resume_context.uc_stack.ss_sp	= calloc(1, MINSIGSTKSZ);
	coro->resume_context.uc_stack.ss_size	= MINSIGSTKSZ;
	coro->resume_context.uc_link		= 0;

	getcontext(&coro->resume_context);
	makecontext(
		&coro->resume_context, 
		(void(*)()) ccoroutine_entry_point, 
		2, 
		coro,
		userdata
	);
	return coro;
}

void *ccoroutine_resume(ccoroutine *coro)
{
	if (coro->is_finished) return (void*)-1;
	swapcontext(&coro->suspend_context, &coro->resume_context);
	return coro->yield_value;
}

void ccoroutine_yield(ccoroutine *coro, void *value)
{
	coro->yield_value = value;
	swapcontext(&coro->resume_context, &coro->suspend_context);
}

void ccoroutine_destroy(ccoroutine *coro)
{
	free(coro->resume_context.uc_stack.ss_sp);
	free(coro);
}
```

#### 测试

```c
void *hello_world(ccoroutine *coro, void *userdata)
{
	printf("%s\n", (char *)userdata);
	ccoroutine_yield(coro, (void *)1);
	printf("END: %s \n", (char *)userdata);
	return (void *)2;
}

void ccoroutine_test()
{
	ccoroutine *coro = ccoroutine_create(hello_world, "hello from 1");
	ccoroutine *coro2 = ccoroutine_create(hello_world, "hello from 2");

	while (!ccoroutine_finished(coro) || !ccoroutine_finished(coro2)) {
		ccoroutine_resume(coro);
		ccoroutine_resume(coro2);
	}

	ccoroutine_destroy(coro);
	ccoroutine_destroy(coro2);
}

int main()
{
	ccoroutine_test();
	return 0;
}
```

## 其他阅读

- [Javascript 中 Promise, Async, Await](https://zhuanlan.zhihu.com/p/148462034)
