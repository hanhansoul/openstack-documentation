# Arguments and results

In TaskFlow, all flow and task state goes to storage. That includes all the information that [atoms](https://docs.openstack.org/taskflow/latest/user/atoms.html) (e.g. tasks, retry objects…) in the workflow need when they are executed, and all the information task/retry produces (via serializable results). A developer who implements tasks/retries or flows can specify what arguments a task/retry accepts and what result it returns in several ways. 

**Task/retry arguments**

Set of names of task/retry arguments available as the `requires` and/or `optional` property of the task/retry instance. When a task or retry object is about to be executed values with these names are retrieved from storage and passed to the `execute` method of the task/retry. If any names in the `requires` property cannot be found in storage, an exception will be thrown. Any names in the `optional` property that cannot be found are ignored.

**Task/retry results**

Set of names of task/retry results (what task/retry provides) available as `provides` property of task or retry instance. After a task/retry finishes successfully, its result(s) (what the `execute` method returns) are available by these names from storage (see examples below).

## Arguments specification

There are different ways to specify the task argument `requires` set.

### Arguments inference

Task/retry arguments can be inferred from arguments of the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute)method of a task (or the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.execute) of a retry object).

```python
class MyTask(task.Task):
	def execute(self, spam, eggs, bacon=None):
		return spam + eggs

sorted(MyTask().requires)
# ['eggs', 'spam']
sorted(MyTask().optional)
# ['bacon']
```

Inference from the method signature is the ‘’simplest’’ way to specify arguments. Special arguments like `self`, `*args` and `**kwargs` are ignored during inference

### Rebinding

**Why:** There are cases when the value you want to pass to a task/retry is stored with a name other than the corresponding arguments name. Using it the flow author can instruct the engine to fetch a value from storage by one name, but pass it to a tasks/retries `execute` method with another name.

 to pass a dictionary that maps the argument name to the name of a saved value.

```
class SpawnVMTask(task.Task):

    def execute(self, vm_name, vm_image_id, **kwargs):
        pass  # TODO(imelnikov): use parameters to spawn vm
```

### Manually specifying requirements

**Why:** It is often useful to manually specify the requirements of a task, either by a task author or by the flow author.

To accomplish this when creating your task use the constructor to specify manual requirements. Those manual requirements (if they are not functional arguments) will appear in the `kwargs` of the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute) method.

```python
class Cat(task.Task):
	def __init__(self, **kwargs):
		if 'requires' not in kwargs:
			kwargs['requires'] = ("food", "milk")
		super(Cat, self).__init__(**kwargs)
		
    def execute(self, food, **kwargs):
		pass

cat = Cat()
sorted(cat.requires)
# ['food', 'milk']
```

When constructing a task instance the flow author can also add more requirements if desired. Those manual requirements (if they are not functional arguments) will appear in the `kwargs` parameter of the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute) method.

```python
class Dog(task.Task):
	def execute(self, food, **kwargs):
		pass
dog = Dog(requires=("water", "grass"))
sorted(dog.requires)
# ['food', 'grass', 'water']
```

If the flow author desires she can turn the argument inference off and override requirements manually. Use this at your own **risk** as you must be careful to avoid invalid argument mappings.

```python
class Bird(task.Task):
    def execute(self, food, **kwargs):
        pass
bird = Bird(requires=("food", "water", "grass"), auto_extract=False)
sorted(bird.requires)
# ['food', 'grass', 'water']
```



## Results specification

In python, function results are not named, so we can not infer what a task/retry returns. This is important since the complete result (what the task [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute) or retry [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.execute) method returns) is saved in (potentially persistent) storage, and it is typically (but not always) desirable to make those results accessible to others. To accomplish this the task/retry specifies names of those values via its `provides` constructor parameter or by its default provides attribute.



#### Returning one value

If task returns just one value, `provides` should be string – the name of the value.

```python
class TheAnswerReturningTask(task.Task):
   def execute(self):
       return 42

sorted(TheAnswerReturningTask(provides='the_answer').provides)
# ['the_answer']
```



#### Returning a tuple

For a task that returns several values, one option (as usual in python) is to return those values via a `tuple`.

```python
class BitsAndPiecesTask(task.Task):
    def execute(self):
        return 'BITs', 'PIECEs'
```

Then, you can give the value individual names, by passing a tuple or list as`provides` parameter:

```python
BitsAndPiecesTask(provides=('bits', 'pieces'))

storage.fetch('bits')
# 'BITs'
storage.fetch('pieces')
# 'PIECEs'
```

Provides argument can be shorter then the actual tuple returned by a task – then extra values are ignored.

#### Returning a dictionary

Another option is to return several values as a dictionary (aka a `dict`).

```python
class BitsAndPiecesTask(task.Task):

    def execute(self):
        return {
            'bits': 'BITs',
            'pieces': 'PIECEs'
        }
```

TaskFlow expects that a dict will be returned if `provides` argument is a `set`:

```python
BitsAndPiecesTask(provides=set(['bits', 'pieces']))
```

After such task executes, you (and the engine, which is useful for other tasks) will be able to get elements from storage by name:

```python
>>> storage.fetch('bits')
'BITs'
>>> storage.fetch('pieces')
'PIECEs'
```

#### Default provides

As mentioned above, the default base class provides nothing, which means results are not accessible to other tasks/retries in the flow.

The author can override this and specify default value for provides using the `default_provides` class/instance variable:

```python
class BitsAndPiecesTask(task.Task):
    default_provides = ('bits', 'pieces')
    def execute(self):
        return 'BITs', 'PIECEs'
```

Of course, the flow author can override this to change names if needed:

```python
BitsAndPiecesTask(provides=('b', 'p'))
```

or to change structure – e.g. this instance will make tuple accessible to other tasks by name `'bnp'`:

```python
BitsAndPiecesTask(provides='bnp')
```

or the flow author may want to return default behavior and hide the results of the task from other tasks in the flow (e.g. to avoid naming conflicts):

```python
BitsAndPiecesTask(provides=())
```



## Revert arguments

To revert a task the [engine](https://docs.openstack.org/taskflow/latest/user/engines.html) calls the tasks [`revert()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.revert) method. This method should accept the same arguments as the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute) method of the task and one more special keyword argument, named `result`.

For `result` value, two cases are possible:

- If the task is being reverted because it failed (an exception was raised from its [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute) method), the `result` value is an instance of a[`Failure`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.failure.Failure) object that holds the exception information.
- If the task is being reverted because some other task failed, and this task finished successfully, `result` value is the result fetched from storage: ie, what the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute) method returned.

All other arguments are fetched from storage in the same way it is done for [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute) method.

To determine if a task failed you can check whether `result` is instance of [`Failure`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.failure.Failure):

```python
from taskflow.types import failure

class RevertingTask(task.Task):

    def execute(self, spam, eggs):
        return do_something(spam, eggs)

    def revert(self, result, spam, eggs):
        if isinstance(result, failure.Failure):
            print("This task failed, exception: %s"
                  % result.exception_str)
        else:
            print("do_something returned %r" % result)
```

If this task failed (ie `do_something` raised an exception) it will print `"Thistask failed, exception:"` and a exception message on revert. If this task finished successfully, it will print `"do_something returned"` and a representation of the `do_something` result.

## Retry arguments

A [`Retry`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry) controller works with arguments in the same way as a [`Task`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.task.Task). But it has an additional parameter `'history'` that is itself a [`History`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.History) object that contains what failed over all the engines attempts (aka the outcomes). The history object can be viewed as a tuple that contains a result of the previous retries run and a table/dict where each key is a failed atoms name and each value is a [`Failure`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.failure.Failure) object.

Consider the following implementation:

```python
class MyRetry(retry.Retry):

    default_provides = 'value'

    def on_failure(self, history, *args, **kwargs):
        print(list(history))
        return RETRY

    def execute(self, history, *args, **kwargs):
        print(list(history))
        return 5

    def revert(self, history, *args, **kwargs):
        print(list(history))
```

Imagine the above retry had returned a value `'5'` and then some task `'A'` failed with some exception. In this case `on_failure` method will receive the following history (printed as a list):

```python
[('5', {'A': failure.Failure()})]
```

At this point (since the implementation returned `RETRY`) the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.execute)method will be called again and it will receive the same history and it can then return a value that subsequent tasks can use to alter their behavior.

If instead the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.execute) method itself raises an exception, the [`revert()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.revert)method of the implementation will be called and a [`Failure`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.failure.Failure) object will be present in the history object instead of the typical result.