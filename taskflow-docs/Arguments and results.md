# 参数与返回值

TaskFlow中，所有Flow和Task的状态信息都会在一个存储空间中被持久化。

状态信息包括：

1. Task/Retry对象执行`execute`方法所需要的信息；
2. Task/Retry对象产生的所有信息。

可以指定Task与Retry的`execute`方法的**参数名称列表**与其**返回值名称列表**。

**Task/Retry参数**

Task/Retry参数名称的集合通过Task/Retry实例的`requires`和`optional`两个列表类型属性指定。当Task或Retry执行时，会从存储空间中获取参数名对应的值，并传递给Task/Retry的`execute`方法。其中`requires`指定了必需参数的参数名，`optional`指定了可选参数的参数名。

**Task/Retry返回值**

Task/Retry返回值名称的集合通过Task/Retry实例的`provides`属性指定。当Task/Retry执行成功时，`execute`方法的返回值可以通过该名称从存储空间中获取。



## 指定参数名称

有多种可以指定Task/Retry的`requires`属性的方法。

### 参数推导

Task/Retry的参数名称列表可以通过`execute`方法的参数列表推导获取。

```python
>>> class MyTask(task.Task):
...     def execute(self, spam, eggs, bacon=None):
...         return spam + eggs
...
>>> sorted(MyTask().requires)
['eggs', 'spam']
>>> sorted(MyTask().optional)
['bacon']

>>> class UniTask(task.Task):
...     def execute(self, *args, **kwargs):
...         pass
...
>>> sorted(UniTask().requires)
[]
```



### 参数重新绑定

当希望Task/Retry的参数名映射到存储空间中不同的名称时，可以通过构造函数的`rebind`参数重新指定。

对于以下Task对象：

```python
class SpawnVMTask(task.Task):

    def execute(self, vm_name, vm_image_id, **kwargs):
        pass  # TODO(imelnikov): use parameters to spawn vm
```

1. 通过字典类型的参数重新指定参数映射关系：

```python
SpawnVMTask(rebind={'vm_name': 'name'})
```

2. 通过长度不小于必需参数个数的tuple/list/dict类型参数指定：

```python
SpawnVMTask(rebind_args=('name', 'vm_image_id'))
# 等价于
SpawnVMTask(rebind=dict(vm_name='name',
                        vm_image_id='vm_image_id'))
```

当task接受一个`**kwargs`参数时，则可以指定可选参数。

```python
SpawnVMTask(rebind=('name', 'vm_image_id', 'admin_key_name'))
```

其中`admin_key_name`为可选参数，传递给`**kwargs`参数。



### 手动指定必需参数

可以在构造函数中通过指定`**kwargs`参数中`requires`的内容，手动指定必需参数。

```python
>>> class Cat(task.Task):
...     def __init__(self, **kwargs):
...         if 'requires' not in kwargs:
...             kwargs['requires'] = ("food", "milk")
...         super(Cat, self).__init__(**kwargs)
...     def execute(self, food, **kwargs):
...         pass
...
>>> cat = Cat()
>>> sorted(cat.requires)
['food', 'milk']
```

可以通过构造函数的`requires`参数添加必需参数，这些手动添加的必需参数会传递给`**kwargs`参数。

```python
>>> class Dog(task.Task):
...     def execute(self, food, **kwargs):
...         pass
...
>>> dog = Dog(requires=("water", "grass"))
>>> sorted(dog.requires)
['food', 'grass', 'water']
```

通过`auto_extract`参数，手动指定参数可以覆盖参数推导的结果。

```python
>>> class Bird(task.Task):
...     def execute(self, food, **kwargs):
...         pass
...
>>> bird = Bird(requires=("food", "water", "grass"), auto_extract=False)
>>> sorted(bird.requires)
['food', 'grass', 'water']
```



## 指定返回值名称

### 返回单个值

```python
>>> class TheAnswerReturningTask(task.Task):
...    def execute(self):
...        return 42
...
>>> sorted(TheAnswerReturningTask(provides='the_answer').provides)
['the_answer']
```

### 返回一个元组

```python
class BitsAndPiecesTask(task.Task):
    def execute(self):
        return 'BITs', 'PIECEs'
        
BitsAndPiecesTask(provides=('bits', 'pieces'))

>>> storage.fetch('bits')
'BITs'
>>> storage.fetch('pieces')
'PIECEs'
```



### 返回一个字典

```python
class BitsAndPiecesTask(task.Task):

    def execute(self):
        return {
            'bits': 'BITs',
            'pieces': 'PIECEs'
        }
        
BitsAndPiecesTask(provides=set(['bits', 'pieces']))

>>> storage.fetch('bits')
'BITs'
>>> storage.fetch('pieces')
'PIECEs'
```



### 默认值



## Revert参数



## Retry参数