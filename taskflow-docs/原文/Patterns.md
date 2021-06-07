# Patterns

```
class taskflow.flow.Flow(name, retry=None)
```

The base abstract class of all flow implementations.

A flow is a structure that defines relationships between tasks. You can add tasks and other flows (as subflows) to the flow, and the flow provides a way to implicitly or explicitly define how they are interdependent. Exact structure of the relationships is defined by concrete implementation, while this class defines common interface and adds human-readable (not necessary unique) name.

```
abstract add(*items)
```

Adds a given item/items to this flow.

```
abstract iter_links()
```

Iterates over dependency links between children of the flow.

Iterates over 3-tuples `(A, B, meta)`, where

- `A` is a child (atom or subflow) link starts from;
- `B` is a child (atom or subflow) link points to; it is said that `B` depends on `A` or `B` requires `A`;
- `meta` is link metadata, a dictionary.

```
abstract iter_nodes()
```

Iterate over nodes of the flow.

Iterates over 2-tuples `(A, meta)`, where

- `A` is a child (atom or subflow) of current flow;
- `meta` is link metadata, a dictionary.



## Linear flow

```python
class taskflow.patterns.linear_flow.Flow(name, retry=None)
```

Linear flow pattern.

A linear (potentially nested) flow of *tasks/flows* that can be applied in order as one unit and rolled back as one unit using the reverse order that the *tasks/flows* have been applied in.



## Unordered flow

```python
class taskflow.patterns.unordered_flow.Flow(name, retry=None)
```

Unordered flow pattern.

A unordered (potentially nested) flow of *tasks/flows* that can be executed in any order as one unit and rolled back as one unit.



## Graph flow

```python
class taskflow.patterns.graph_flow.Flow(name, retry=None)
```

Graph flow pattern.

Contained *flows/tasks* will be executed according to their dependencies which will be resolved by using the *flows/tasks* provides and requires mappings or by following manually created dependency links.

From dependencies a [directed graph](https://en.wikipedia.org/wiki/Directed_graph) is built. If it has edge `A -> B`, this means `B` depends on `A` (and that the execution of `B` must wait until `A` has finished executing, on reverting this means that the reverting of `A` must wait until `B` has finished reverting).

Note: [cyclic](https://en.wikipedia.org/wiki/Cycle_graph) dependencies are not allowed.

```
link(u, v, decider=None, decider_depth=None)
```

Link existing node u as a runtime dependency of existing node v.

Note that if the addition of these edges creates a [cyclic](https://en.wikipedia.org/wiki/Cycle_graph) graph then a [`DependencyFailure`](https://docs.openstack.org/taskflow/latest/user/exceptions.html#taskflow.exceptions.DependencyFailure) will be raised and the provided changes will be discarded. If the nodes that are being requested to link do not exist in this graph than a `ValueError`will be raised.



```
add(*nodes, **kwargs)
```

Adds a given task/tasks/flow/flows to this flow.

Note that if the addition of these nodes (and any edges) creates a [cyclic](https://en.wikipedia.org/wiki/Cycle_graph) graph then a [`DependencyFailure`](https://docs.openstack.org/taskflow/latest/user/exceptions.html#taskflow.exceptions.DependencyFailure) will be raised and the applied changes will be discarded.



```
class taskflow.patterns.graph_flow.TargetedFlow(*args, **kwargs)
```

Graph flow with a target.

Adds possibility to execute a flow up to certain graph node (task or subflow).



```python
class taskflow.deciders.Depth(value)
```

Enumeration of decider(s) *area of influence*.

`ALL` *= 'ALL'*

**Default** decider depth that affects **all** successor atoms (including ones that are in successor nested flows).

`FLOW` *= 'FLOW'*

Decider depth that affects **all** successor tasks in the **same** flow (it will **not** affect tasks/retries that are in successor nested flows).

`NEIGHBORS` *= 'NEIGHBORS'*

Decider depth that affects only **next** successor tasks (and does not traverse past **one**level of successor tasks).

`ATOM` *= 'ATOM'*

Decider depth that affects only **targeted** atom (and does **not** traverse into **any** level of successor atoms).



## Hierarchy

![Inheritance diagram of taskflow.flow, taskflow.patterns.linear_flow, taskflow.patterns.unordered_flow, taskflow.patterns.graph_flow](https://docs.openstack.org/taskflow/latest/_images/inheritance-ccbc00437ef6376ea2da60181d9c2f76b6b989fb.png)