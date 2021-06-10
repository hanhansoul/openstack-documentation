# Flow的输入与输出

一般而言，Task通过Task的输入参数接收输入，通过Task的返回值提供输出。这是最常见的一种将数据从一个Task传递给另一个Task的方法。

1. Flow的输入参数名列表：通过Flow的`requires`属性获取。若一个Task的某些必需输入参数的参数名并不是任何一个Task的输出参数的参数名，则该参数为Flow的输入参数。由于没有任何Task会返回该参数的值，Flow的输入参数必须在Flow运行前预存储在存储空间中以备获取。

2. Flow的输出参数名列表：通过Flow的`provides`属性获取。Flow的输出参数名列表由Flow中所有Task的输出参数名列表取并集组成。

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



# 引擎与存储空间

引擎通过存储层实现Flow与Task数据的持久化。

## Flow的输入参数

如果在运行时，获取Flow的输入参数失败，则会抛出异常。

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

可以通过Engine的`run()`或`load()`方法的`store`输入参数提供Flow的输入参数，将其预存储在存储空间中。

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



## Flow的输出参数

获取Flow的输出参数的方法：

1. 通过Engine的`run()`方法返回。
2. 通过`fetch_all()`方法返回。
3. 通过`fetch()`方法返回。

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



---



```
class taskflow.flow.Flow(name, retry=None)

class taskflow.patterns.linear_flow.Flow(name, retry=None)
```

