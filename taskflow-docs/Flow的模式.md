# Flow的模式

```python
class taskflow.flow.Flow(name, retry=None)
```

`Flow`是所有Flow的基类。

Flow是一种描述Task之间关系的结构。通过向Flow中添加Task或子Flow，可以定义它们之间的依赖关系。

1. `abstract add(*items)`：将给定的Task或Flow添加到当前Flow中。

2. `abstract iter_links()`：遍历Flow与其儿子节点的依赖连接关系。
3. `abstract iter_nodes()`：遍历Flow的节点。

## 线性Flow模式

```python
class taskflow.patterns.linear_flow.Flow(name, retry=None)
```

线性Flow模式：线性Flow模式中的多个Task或子Flow会作为一个整体依次执行或失败回滚。

## 无序Flow模式

```python
class taskflow.patterns.unordered_flow.Flow(name, retry=None)
```

无序Flow模式：无序Flow模式中的多个Task或子Flow会作为一个整体以任意顺序执行或失败回滚。

## 图Flow模式

```python
class taskflow.patterns.graph_flow.Flow(name, retry=None)
```

图Flow模式：根据Task与Flow的关系构建一个有向图。Task与Flow会根据互相之间的依赖关系依次执行。

```python
class taskflow.patterns.graph_flow.TargetedFlow(*args, **kwargs)
```

有目标节点的图Flow模式：

## 继承关系图

![Inheritance diagram of taskflow.flow, taskflow.patterns.linear_flow, taskflow.patterns.unordered_flow, taskflow.patterns.graph_flow](https://docs.openstack.org/taskflow/latest/_images/inheritance-ccbc00437ef6376ea2da60181d9c2f76b6b989fb.png)