# Persistence

## Overview

In order to be able to receive inputs and create outputs from atoms (or other engine processes) in a fault-tolerant way, there is a need to be able to place what atoms output in some kind of location where it can be re-used by other atoms (or used for other purposes). To accommodate this type of usage TaskFlow provides an abstraction that is similar in concept to a running programs *memory*.

为了实现Atom在接收与返回参数时的容错性，需要将Atom的返回值持久化处理。

This abstraction serves the following *major* purposes:

- Tracking of what was done (introspection).
- Saving *memory* which allows for restarting from the last saved state which is a critical feature to restart and resume workflows (checkpointing).
- Associating additional metadata with atoms while running (without having those atoms need to save this data themselves). This makes it possible to add-on new metadata in the future without having to change the atoms themselves. For example the following can be saved:
  - Timing information (how long a task took to run).
  - User information (who the task ran as).
  - When a atom/workflow was ran (and why).
- Saving historical data (failures, successes, intermediary results…) to allow for retry atoms to be able to decide if they should should continue vs. stop.
- *Something you create…*

## How it is used

On [engine](https://docs.openstack.org/taskflow/latest/user/engines.html) construction typically a backend (it can be optional) will be provided which satisfies the [`Backend`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.base.Backend) abstraction. Along with providing a backend object a [`FlowDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.FlowDetail) object will also be created and provided (this object will contain the details about the flow to be ran) to the engine constructor (or associated [`load()`](https://docs.openstack.org/taskflow/latest/user/engines.html#taskflow.engines.helpers.load) helper functions). Typically a [`FlowDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.FlowDetail) object is created from a [`LogBook`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.LogBook) object (the book object acts as a type of container for [`FlowDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.FlowDetail) and [`AtomDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.AtomDetail) objects).

Engine对象创建时，会创建一个Backend对象提供存储功能。同时，LogBook对象会创建一个包含Flow信息的FlowDetail对象，并通过构造函数或`load()`函数传递给Engine对象。

LogBook对象可以看做时FlowDetail和AtomDetail对象的一种容器。

**Preparation**: Once an engine starts to run it will create a [`Storage`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.storage.Storage) object which will act as the engines interface to the underlying backend storage objects (it provides helper functions that are commonly used by the engine, avoiding repeating code when interacting with the provided[`FlowDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.FlowDetail) and [`Backend`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.base.Backend) objects). As an engine initializes it will extract (or create) [`AtomDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.AtomDetail)objects for each atom in the workflow the engine will be executing.

准备：Engine对象在开始运行时，会创建一个Storage对象，作为一个接口来访问Backend对象中的FlowDetail对象。在初始化Storage对象时，Engine会为每一个Atom创建一个AtomDetail对象。

**Execution:** When an engine begins to execute (see [engine](https://docs.openstack.org/taskflow/latest/user/engines.html) for more of the details about how an engine goes about this process) it will examine any previously existing [`AtomDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.AtomDetail) objects to see if they can be used for resuming; see [resumption](https://docs.openstack.org/taskflow/latest/user/resumption.html) for more details on this subject. For atoms which have not finished (or did not finish correctly from a previous run) they will begin executing only after any dependent inputs are ready. This is done by analyzing the execution graph and looking at predecessor [`AtomDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.AtomDetail) outputs and states (which may have been persisted in a past run). This will result in either using their previous information or by running those predecessors and saving their output to the [`FlowDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.FlowDetail) and [`Backend`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.base.Backend) objects. This execution, analysis and interaction with the storage objects continues (what is described here is a simplification of what really happens; which is quite a bit more complex) until the engine has finished running (at which point the engine will have succeeded or failed in its attempt to run the workflow).

执行：在Engine开始执行时，会检查每一个之前存在的AtomDetail对象的状态，来确定是否执行对应的Atom。

**Post-execution:** Typically when an engine is done running the logbook would be discarded (to avoid creating a stockpile of useless data) and the backend storage would be told to delete any contents for a given execution. For certain use-cases though it may be advantageous to retain logbooks and their contents.

执行后：Engine执行完成后，Logbook以及Backend中存储的信息会被销毁，但在以下情况会保留其中的信息：

- Post runtime failure analysis and triage (saving what failed and why).
- Metrics (saving timing information associated with each atom and using it to perform offline performance analysis, which enables tuning tasks and/or isolating and fixing slow tasks).
- Data mining logbooks to find trends (in failures for example).
- Saving logbooks for further forensics analysis.
- Exporting logbooks to [hdfs](https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html) (or other no-sql storage) and running some type of map-reduce jobs on them.

## Usage

To select which persistence backend to use you should use the [`fetch()`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.backends.fetch) function which uses entrypoints (internally using [stevedore](https://docs.openstack.org/stevedore/latest/)) to fetch and configure your backend. This makes it simpler than accessing the backend data types directly and provides a common function from which a backend can be fetched.

通过`taskflow.persistence.backends.fetch()`函数，能够获取并配置Backend。

```python
from taskflow.persistence import backends

...
persistence = backends.fetch(conf={
    "connection": "mysql",
    "user": ...,
    "password": ...,
})
book = make_and_save_logbook(persistence)
...
```



## Types

### Memory

**Connection**: `'memory'`

`MemoryBackend`

### Files

**Connection**: `'dir'` or `'file'`

`DirBackend`

### SQLAlchemy

**Connection**: `'mysql'` or `'postgres'` or `'sqlite'`

`SQLAlchemyBackend`

#### Schema

*Logbooks*

| Name       | Type     | Primary Key |
| :--------- | :------- | :---------- |
| created_at | DATETIME | False       |
| updated_at | DATETIME | False       |
| uuid       | VARCHAR  | True        |
| name       | VARCHAR  | False       |
| meta       | TEXT     | False       |

*Flow details*

| Name        | Type     | Primary Key |
| :---------- | :------- | :---------- |
| created_at  | DATETIME | False       |
| updated_at  | DATETIME | False       |
| uuid        | VARCHAR  | True        |
| name        | VARCHAR  | False       |
| meta        | TEXT     | False       |
| state       | VARCHAR  | False       |
| parent_uuid | VARCHAR  | False       |

*Atom details*

| Name        | Type     | Primary Key |
| :---------- | :------- | :---------- |
| created_at  | DATETIME | False       |
| updated_at  | DATETIME | False       |
| uuid        | VARCHAR  | True        |
| name        | VARCHAR  | False       |
| meta        | TEXT     | False       |
| atom_type   | VARCHAR  | False       |
| state       | VARCHAR  | False       |
| intention   | VARCHAR  | False       |
| results     | TEXT     | False       |
| failure     | TEXT     | False       |
| version     | TEXT     | False       |
| parent_uuid | VARCHAR  | False       |



### Zookeeper

**Connection**: `'zookeeper'`

`ZkBackend`



```python
# 根据给定的配置项获取backend。
taskflow.persistence.backends.fetch(conf, namespace='taskflow.persistence', **kwargs)

taskflow.persistence.backends.backend(conf, namespace='taskflow.persistence', **kwargs)

# Backend的基类
class taskflow.persistence.base.Backend(conf)

# Backend Connection的基类
class taskflow.persistence.base.Connection

# Backend的具体实现
class taskflow.persistence.path_based.PathBasedBackend(conf)
# Backend Connection的具体实现
class taskflow.persistence.path_based.PathBasedConnection(backend)

# Model
# FlowDetail和关联元数据
class taskflow.persistence.models.LogBook(name, uuid=None)

# AtomDetail和关联元数据
class taskflow.persistence.models.FlowDetail(name, uuid)

# Atom运行时详情信息与元数据的集合
class taskflow.persistence.models.AtomDetail(name, uuid)
# 继承了AtomDetail，包含了一个Task的详情信息
class taskflow.persistence.models.TaskDetail(name, uuid)
# 继承了AtomDetail，包含了一个Retry的详情信息
class taskflow.persistence.models.RetryDetail(name, uuid)

# Implementations
# In-memory
class taskflow.persistence.backends.impl_memory.FakeInode(item, path, value=None)
class taskflow.persistence.backends.impl_memory.FakeFilesystem(deep_copy=True)
class taskflow.persistence.backends.impl_memory.MemoryBackend(conf=None)
class taskflow.persistence.backends.impl_memory.Connection(backend)

# File
class taskflow.persistence.backends.impl_dir.DirBackend(conf)
class taskflow.persistence.backends.impl_dir.Connection(backend)

# SQLAlchemy
class taskflow.persistence.backends.impl_sqlalchemy.SQLAlchemyBackend(conf, engine=None)
class taskflow.persistence.backends.impl_sqlalchemy.Connection(backend, upgrade_lock)

# Zookeeper
class taskflow.persistence.backends.impl_zookeeper.ZkBackend(conf, client=None)
class taskflow.persistence.backends.impl_zookeeper.ZkConnection(backend, client, conf)

# Storage
# Engine与Logbook（及其Backend）之间的接口
# 为Engine提供保存Atom和Flow相关信息到持久层的功能
class taskflow.storage.Storage(flow_detail, backend=None, scope_fetcher=None)
```



## Hierarchy

![Inheritance diagram of taskflow.persistence.base, taskflow.persistence.backends.impl_dir, taskflow.persistence.backends.impl_memory, taskflow.persistence.backends.impl_sqlalchemy, taskflow.persistence.backends.impl_zookeeper](https://docs.openstack.org/taskflow/latest/_images/inheritance-df83794bd029d44d738f25307d7c2e2180498899.png)



