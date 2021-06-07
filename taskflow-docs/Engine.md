# Engine

## 概述

Engine负责TaskFlow中Task等Atom元素的实际运行。

Engine接受一个Flow作为输入，并据此确定Atom元素的执行顺序。

## 动机

1. 命令式编程：命令“机器”如何去做事情(how)，这样不管你想要的是什么(what)，它都会按照你的命令实现。命令式编程的主要思想是关注计算机执行的步骤， **即一步一步告诉计算机先做什么再做什么。**
2. 声明式编程：告诉“机器”你想要的是什么(what)，让机器想出如何去做(how)。声明式编程是以**数据结构**的形式来表达程序执行的逻辑。 **它的主要思想是告诉计算机应该做什么，但不指定具体要怎么做。**

如果要解决以下一个问题：求一个list里所有值的和。

命令式编程：

```python
list = [1,2,3,4,5];
total = 0;

for x in list:
	total += x
	
print(total)
```

声明式编程：

```python
list = [1,2,3,4,5];

total = reduce(lambda x, y: x + y, list)

print(total)
```

声明式编程的优势在于：开发者只需要“声明”需要做什么，而具体怎么做，则由编程语言本身在运行时去解决，这极大地提高了开发的效率。

## TaskFlow的编程模式

TaskFlow中：

1. What：Flow的结构与Flow中包含的Task等Atom元素；
2. How：未定义。

特性：

1. 可靠性
2. 扩展性
3. 一致性

## 创建

通常使用Engine的辅助方法来创建一个Engine。

```python
from taskflow import engines

...
flow = make_flow()
eng = engines.load(flow, engine='serial', backend=my_persistence_conf)
eng.run()
...
```

```python
taskflow.engines.helpers.load(flow, 
							store=None, 
							flow_detail=None, 
							book=None, 
							backend=None, 
							namespace='taskflow.engines', 
							engine='default', 
							**kwargs)
```

将一个Flow装载进一个Engine中。



```python
taskflow.engines.helpers.run(flow, 
							store=None, 
							flow_detail=None, 
							book=None, 
							backend=None, 
							namespace='taskflow.engines', 
							engine='default', 
							**kwargs)
```



## 类型

### Serial

所有Task都在一个线程中运行，即调用`run()`的线程。serial类型为Engine的默认类型。

### Parallel

`ParallelActionEngine`：Task将在不同的线程或进程中进行调度并执行。

### Workers

略。



## 运行过程

1. 创建

2. 编译`Engine.compile()`：在编译阶段，Engine中加载的Flow会被转换为一个networkx有向图或树，包含了Flow运行的所有信息。编译结束时，编译器将返回一个包含了所有的运行时组件引用的`Runtime`对象，一些运行时必须的辅助对象也将被创建并存储在Engine对象的变量中。

3. 准备`Engine.prepare()`：配置存储空间，并创建Flow中每个节点对应的`AtomDetail`对象。

4. 校验`Engine.validate()`：对当前已经准备就绪的Engine对象进行校验。

5. 执行`Engine.run()`：Flow的状态变为`RUNNING`，状态机`MachineBUilder`和运行对象通过`automaton`类库被创建，并开始控制执行以下阶段。
   1. 重启：通过Task的TaskDetail对象，分析Flow中Task的状态，并根据策略与Flow的结构确定如何调度Task的执行。
   2. 调度：通过`Scheduler`调用Task的执行，`Scheduler`会为每一个被调度指定的Atom返回一个`Future`对象，之后进入下一阶段。
   3. 等待：Engine会等待所有`Future`对象完成，一旦某个Future完成，将调用`Completer`，并通过`Storage`持久化到对应的存储空间中，即`AtomDetail`或`FlowDetail`对象。根据Atom的执行是否成功，Engine会设置Atom的状态。
6. 结束：根据执行的结果，Flow会被置为`FAILURE`, `SUSPENDED`, `SUCCESS`或 `REVERTED`其中一种状态。
7. 挂起`suspend()`：特殊场景。`suspend()`方法会从外部请求挂起当前任务，Flow的状态由`RUNNING`变为`SUSPENDING`。通过`run()`可以恢复当前任务。



## 作用域

### 作用域解析的默认策略

1. Transient injected atom specific arguments.
2. Non-transient injected atom specific arguments.
3. Transient injected arguments (flow specific).
4. Non-transient injected arguments (flow specific).
5. First scope visited provider that produces the named result; note that if multiple providers are found in the same scope the first (the scope walkers yielded ordering defines what first means) that produced that result and can be extracted without raising an error is selected as the provider of the requested requirement.
6. Fails with NotFound if unresolved at this point (the cause attribute of this exception may have more details on why the lookup failed).



---

Engine提供了多个不同的辅助函数，用于创建、加载、运行Flow。

```python
# 将一个Flow装载到一个Engine中
taskflow.engines.helpers.load(flow, 
                              store=None, 
                              flow_detail=None, 
                              book=None, 
                              backend=None, 
                              namespace='taskflow.engines', 
                              engine='default', 
                              **kwargs)

# 运行载入的Flow。
taskflow.engines.helpers.run(flow, 
                             store=None, 
                             flow_detail=None, 
                             book=None, 
                             backend=None, 
                             namespace='taskflow.engines', 
                             engine='default', 
                             **kwargs)

# 将指定的Flow工厂对象详情信息存储到FlowDetail对象中
taskflow.engines.helpers.save_factory_details(flow_detail, 
                                             flow_factory, 
                                             factory_args, 
                                             factory_kwargs, 
                                             backend=None)

# 通过工厂方法创建一个Flow，并装载到Engine中
taskflow.engines.helpers.load_from_factory(flow_factory, 
                                           factory_args=None, 
                                           factory_kwargs=None, 
                                           store=None, 
                                           book=None, 
                                           backend=None, 
                                           namespace='taskflow.engines',
                                           engine='default', 
                                           **kwargs)

taskflow.engines.helpers.flow_from_detail(flow_detail)

taskflow.engines.helpers.load_from_detail(flow_detail, 
                                          store=None, 
                                          backend=None, 
                                          namespace='taskflow.engines', 
                                          engine='default', 
                                          **kwargs)
```

