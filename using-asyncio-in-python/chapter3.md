## 协程

### `async def`关键字

`async def`函数返回一个协程对象。



1. 协程通过`send(None)`进行预激活。事件循环能够实现协程的预激活，而不需要手动执行。所有的协程可以通过`loop.create_task(coro)`或`await coro`调用执行。
2. 当协程返回时，会抛出`StopIteration`异常，可以通过异常的`value`属性获取协程的返回值。`async def`函数只需要像一般函数一样返回结果即可。



协程中`send(None)`或`StopIteration`分别是执行协程开始与结束的标识。事件循环能够在底层自动驱动协程的执行，开发者只需要为协程执行，事件循环会像调用普通函数一样自顶向下调用协程。



### `await`关键字



---

1. 通过`async def`声明一个协程函数，调用协程函数会返回一个协程对象。
2. 通过`iscoroutinefunction()`函数区分普通函数与协程函数，通过`async def`声明的函数即协程函数。
3. 通过`iscoroutine()`函数判断一个对象是否为协程对象。
4. 



## 事件循环

Asyncio中的事件循环负责处理协程间的切换，捕获`StopIteration`异常，并且监听socket或文件IO事件。

获取事件循环的方式：

1. `asyncio.get_running_loop()`：必须在一个协程上下文内部调用。
2. `asyncio.get_event_loop()`：可以在任意位置调用。