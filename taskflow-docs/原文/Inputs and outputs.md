# Inputs and outputs

In TaskFlow there are multiple ways to provide inputs for your tasks and flows and get information from them.

## Flow inputs and outputs

Tasks accept inputs via task arguments and provide outputs via task results. This is the standard and recommended way to pass data from one task to another. Of course not every task argument needs to be provided to some other task of a flow, and not every task result should be consumed by every task.

If some value is required by one or more tasks of a flow, but it is not provided by any task, it is considered to be flow input, and **must** be put into the storage before the flow is run. A set of names required by a flow can be retrieved via that flow’s `requires` property. These names can be used to determine what names may be applicable for placing in storage ahead of time and which names are not applicable.

All values provided by tasks of the flow are considered to be flow outputs; the set of names of such values is available via the `provides` property of the flow.

For example:

```python
>>> class MyTask(task.Task):
...     def execute(self, **kwargs):
...         return 1, 2
...
>>> flow = linear_flow.Flow('test').add(
...     MyTask(requires='a', provides=('b', 'c')),
...     MyTask(requires='b', provides='d')
... )
>>> flow.requires
frozenset(['a'])
>>> sorted(flow.provides)
['b', 'c', 'd']
```

## Engine and storage

The storage layer is how an engine persists flow and task details.

### Inputs

As mentioned above, if some value is required by one or more tasks of a flow, but is not provided by any task, it is considered to be flow input, and **must** be put into the storage before the flow is run. On failure to do so [`MissingDependencies`](https://docs.openstack.org/taskflow/latest/user/exceptions.html#taskflow.exceptions.MissingDependencies) is raised by the engine prior to running:

```python
>>> class CatTalk(task.Task):
...   def execute(self, meow):
...     print meow
...     return "cat"
...
>>> class DogTalk(task.Task):
...   def execute(self, woof):
...     print woof
...     return "dog"
...
>>> flo = linear_flow.Flow("cat-dog")
>>> flo.add(CatTalk(), DogTalk(provides="dog"))
<taskflow.patterns.linear_flow.Flow object at 0x...>
>>> engines.run(flo)
Traceback (most recent call last):
   ...
taskflow.exceptions.MissingDependencies: 'linear_flow.Flow: cat-dog(len=2)' requires ['meow', 'woof'] but no other entity produces said requirements
 MissingDependencies: 'execute' method on '__main__.DogTalk==1.0' requires ['woof'] but no other entity produces said requirements
 MissingDependencies: 'execute' method on '__main__.CatTalk==1.0' requires ['meow'] but no other entity produces said requirements
```

The recommended way to provide flow inputs is to use the `store` parameter of the engine helpers

```python
>>> class CatTalk(task.Task):
...   def execute(self, meow):
...     print meow
...     return "cat"
...
>>> class DogTalk(task.Task):
...   def execute(self, woof):
...     print woof
...     return "dog"
...
>>> flo = linear_flow.Flow("cat-dog")
>>> flo.add(CatTalk(), DogTalk(provides="dog"))
<taskflow.patterns.linear_flow.Flow object at 0x...>
>>> result = engines.run(flo, store={'meow': 'meow', 'woof': 'woof'})
meow
woof
>>> pprint(result)
{'dog': 'dog', 'meow': 'meow', 'woof': 'woof'}
```

You can also directly interact with the engine storage layer to add additional values, note that if this route is used you can’t use the helper method [`run()`](https://docs.openstack.org/taskflow/latest/user/engines.html#taskflow.engines.helpers.run). Instead, you must activate the engine’s run method directly `run()`:

```python
>>> flo = linear_flow.Flow("cat-dog")
>>> flo.add(CatTalk(), DogTalk(provides="dog"))
<taskflow.patterns.linear_flow.Flow object at 0x...>
>>> eng = engines.load(flo, store={'meow': 'meow'})
>>> eng.storage.inject({"woof": "bark"})
>>> eng.run()
meow
bark
```

### Outputs

As you can see from examples above, the run method returns all flow outputs in a `dict`. This same data can be fetched via [`fetch_all()`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.storage.Storage.fetch_all) method of the engines storage object. You can also get single results using the engines storage objects [`fetch()`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.storage.Storage.fetch) method.

For example:

```python
>>> eng = engines.load(flo, store={'meow': 'meow', 'woof': 'woof'})
>>> eng.run()
meow
woof
>>> pprint(eng.storage.fetch_all())
{'dog': 'dog', 'meow': 'meow', 'woof': 'woof'}
>>> print(eng.storage.fetch("dog"))
dog
```