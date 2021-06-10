# Notifications and listeners

Engines provide a way to receive notification on task and flow state transitions, which is useful for monitoring, logging, metrics, debugging and plenty of other tasks.

To receive these notifications you should register a callback with an instance of the [`Notifier`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.notifier.Notifier)class that is attached to [`Engine`](https://docs.openstack.org/taskflow/latest/user/engines.html#taskflow.engines.base.Engine) attributes `atom_notifier` and `notifier`.

TaskFlow also comes with a set of predefined [listeners](https://docs.openstack.org/taskflow/latest/user/notifications.html#listeners), and provides means to write your own listeners, which can be more convenient than using raw callbacks.

Engine提供了在Task或Flow状态变化接收通知的功能，可用于监控、日志、指标统计及调试。

通过为Engine注册一个`Notifier`实例的回调方法，可实现通知功能。



## Receiving notifications with callbacks

### Flow notifications

To receive notification on flow state changes use the [`Notifier`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.notifier.Notifier) instance available as the `notifier` property of an engine.

```python
>>> class CatTalk(task.Task):
...   def execute(self, meow):
...     print(meow)
...     return "cat"
...
>>> class DogTalk(task.Task):
...   def execute(self, woof):
...     print(woof)
...     return 'dog'
...
>>> def flow_transition(state, details):
...     print("Flow '%s' transition to state %s" % (details['flow_name'], state))
...
>>>
>>> flo = linear_flow.Flow("cat-dog").add(CatTalk(), DogTalk(provides="dog"))
>>> eng = engines.load(flo, store={'meow': 'meow', 'woof': 'woof'})
>>> eng.notifier.register(ANY, flow_transition)
>>> eng.run()
Flow 'cat-dog' transition to state RUNNING
meow
woof
Flow 'cat-dog' transition to state SUCCESS
```



### Task notifications

To receive notification on task state changes use the [`Notifier`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.notifier.Notifier) instance available as the`atom_notifier` property of an engine.

```python
>>> class CatTalk(task.Task):
...   def execute(self, meow):
...     print(meow)
...     return "cat"
...
>>> class DogTalk(task.Task):
...   def execute(self, woof):
...     print(woof)
...     return 'dog'
...
>>> def task_transition(state, details):
...     print("Task '%s' transition to state %s" % (details['task_name'], state))
...
>>>
>>> flo = linear_flow.Flow("cat-dog")
>>> flo.add(CatTalk(), DogTalk(provides="dog"))
<taskflow.patterns.linear_flow.Flow object at 0x...>
>>> eng = engines.load(flo, store={'meow': 'meow', 'woof': 'woof'})
>>> eng.atom_notifier.register(ANY, task_transition)
>>> eng.run()
Task 'CatTalk' transition to state RUNNING
meow
Task 'CatTalk' transition to state SUCCESS
Task 'DogTalk' transition to state RUNNING
woof
Task 'DogTalk' transition to state SUCCESS
```



## Listeners

TaskFlow comes with a set of predefined listeners – helper classes that can be used to do various actions on flow and/or tasks transitions. You can also create your own listeners easily, which may be more convenient than using raw callbacks for some use cases.

```python
>>> from taskflow.listeners import printing
>>> class CatTalk(task.Task):
...   def execute(self, meow):
...     print(meow)
...     return "cat"
...
>>> class DogTalk(task.Task):
...   def execute(self, woof):
...     print(woof)
...     return 'dog'
...
>>>
>>> flo = linear_flow.Flow("cat-dog").add(
...   CatTalk(), DogTalk(provides="dog"))
>>> eng = engines.load(flo, store={'meow': 'meow', 'woof': 'woof'})
>>> with printing.PrintingListener(eng):
...   eng.run()
...
<taskflow.engines.action_engine.engine.SerialActionEngine object at ...> has moved flow 'cat-dog' (...) into state 'RUNNING' from state 'PENDING'
<taskflow.engines.action_engine.engine.SerialActionEngine object at ...> has moved task 'CatTalk' (...) into state 'RUNNING' from state 'PENDING'
meow
<taskflow.engines.action_engine.engine.SerialActionEngine object at ...> has moved task 'CatTalk' (...) into state 'SUCCESS' from state 'RUNNING' with result 'cat' (failure=False)
<taskflow.engines.action_engine.engine.SerialActionEngine object at ...> has moved task 'DogTalk' (...) into state 'RUNNING' from state 'PENDING'
woof
<taskflow.engines.action_engine.engine.SerialActionEngine object at ...> has moved task 'DogTalk' (...) into state 'SUCCESS' from state 'RUNNING' with result 'dog' (failure=False)
<taskflow.engines.action_engine.engine.SerialActionEngine object at ...> has moved flow 'cat-dog' (...) into state 'SUCCESS' from state 'RUNNING'
```



## Hierarchy



![Inheritance diagram of taskflow.listeners.base.DumpingListener, taskflow.listeners.base.Listener, taskflow.listeners.capturing.CaptureListener, taskflow.listeners.claims.CheckingClaimListener, taskflow.listeners.logging.DynamicLoggingListener, taskflow.listeners.logging.LoggingListener, taskflow.listeners.printing.PrintingListener, taskflow.listeners.timing.PrintingDurationListener, taskflow.listeners.timing.EventTimeListener, taskflow.listeners.timing.DurationListener](https://docs.openstack.org/taskflow/latest/_images/inheritance-3cf6584e8965804e986a5d2b6718af2415e680cb.png)



### Implementations

#### 日志监听器

```python
class taskflow.listeners.logging.LoggingListener(engine, 
                                                 task_listen_for=('*'), 
                                                 flow_listen_for=('*'), 
                                                 retry_listen_for=('*'), 
                                                 log=None, 
                                                 level=10)
```

监听Task或Flow的通知，并记录到给定的日志中，默认使用(`taskflow.listeners.logging`日志。

```python
class taskflow.listeners.logging.DynamicLoggingListener(engine, 
                                                        task_listen_for=('*'),
                                                        flow_listen_for=('*'),
                                                        retry_listen_for=('*'), 
                                                        log=None, 
                                                        failure_level=30, 
                                                        level=10, 
                                                        hide_inputs_outputs_of=(),
                                                        fail_formatter=None)
```

同上。可配置输出日志级别。

```python
class taskflow.listeners.printing.PrintingListener(engine, 
                                                   task_listen_for=('*'), 
                                                   flow_listen_for=('*'), 
                                                   retry_listen_for=('*'), 
                                                   stderr=False)
```

监听Task或Flow通知，将通知信息输出到`stdout`或`stderr`。

#### 时间监听器

```python
class taskflow.listeners.timing.DurationListener(engine)
```

监听并返回Task运行时间。

```python
class taskflow.listeners.timing.PrintingDurationListener(engine, printer=None)
```

同上。记录并输出Task的运行时间。

```python
class taskflow.listeners.timing.EventTimeListener(engine, 
                                                  task_listen_for=('*'), 
                                                  flow_listen_for=('*'), 
                                                  retry_listen_for=('*'))
```

监听Task，Flow和Retry的事件时间戳。



```python
lass taskflow.listeners.claims.CheckingClaimListener(engine, 
                                                     job, 
                                                     board, 
                                                     owner, 
                                                     on_job_loss=None)
```



```python
class taskflow.listeners.capturing.CaptureListener(engine, 
                                                   task_listen_for=('*'), 
                                                   flow_listen_for=('*'), 
                                                   retry_listen_for=('*'), 
                                                   capture_flow=True, 
                                                   capture_task=True, 
                                                   capture_retry=True, 
                                                   skip_tasks=None, 
                                                   skip_retries=None, 
                                                   kip_flows=None, 
                                                   values=None)
```

监听并记录状态的变化。主要用于测试。



#### 格式化器

```python
class taskflow.formatters.FailureFormatter(engine, hide_inputs_outputs_of=())
```

格式化返回的失败信息。