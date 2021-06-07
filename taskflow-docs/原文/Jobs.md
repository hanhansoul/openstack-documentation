# Jobs

## Overview

Jobs and jobboards are a **novel** concept that TaskFlow provides to allow for automatic ownership transfer of workflows between capable owners (those owners usually then use [engines](https://docs.openstack.org/taskflow/latest/user/engines.html) to complete the workflow). They provide the necessary semantics to be able to atomically transfer a job from a producer to a consumer in a reliable and fault tolerant manner. They are modeled off the concept used to post and acquire work in the physical world (typically a job listing in a newspaper or online website serves a similar role).

TaskFlow通过一种高可靠并高容错的方式将job从生产者传递给消费者。

**TLDR:** It’s similar to a queue, but consumers lock items on the queue when claiming them, and only remove them from the queue when they’re done with the work. If the consumer fails, the lock is *automatically* released and the item is back on the queue for further consumption.



## Definitions

**Jobs**

A [`job`](https://docs.openstack.org/taskflow/latest/user/jobs.html#taskflow.jobs.base.Job) consists of a unique identifier, name, and a reference to a [`logbook`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.LogBook) which contains the details of the work that has been or should be/will be completed to finish the work that has been created for that job.

Job由一个唯一标识、一个名称和一个包含任务详情信息的logbook引用组成。

**Jobboards**

A [`jobboard`](https://docs.openstack.org/taskflow/latest/user/jobs.html#taskflow.jobs.base.JobBoard) is responsible for managing the posting, ownership, and delivery of jobs. It acts as the location where jobs can be posted, claimed and searched for; typically by iteration or notification. Jobboards may be backed by different *capable* implementations (each with potentially differing configuration) but all jobboards implement the same interface and semantics so that the backend usage is as transparent as possible. This allows deployers or developers of a service that uses TaskFlow to select a jobboard implementation that fits their setup (and their intended usage) best.

Jobborad负责job的发布、归属和转移的管理。

## High level architecture

![../_images/jobboard.png](https://docs.openstack.org/taskflow/latest/_images/jobboard.png)

**Note:** This diagram shows the high-level diagram (and further parts of this documentation also refer to it as well) of the zookeeper implementation (other implementations will typically have different architectures).

基于Zookeeper的Jobboard实现。

## Features

- High availability
  - Guarantees workflow forward progress by transferring <u>partially complete work</u> or <u>work that has not been started</u> to entities which can either resume the previously partially completed work or begin initial work to ensure that the workflow as a whole progresses (where progressing implies transitioning through the workflow [patterns](https://docs.openstack.org/taskflow/latest/user/patterns.html) and [atoms](https://docs.openstack.org/taskflow/latest/user/atoms.html) and completing their associated [states](https://docs.openstack.org/taskflow/latest/user/states.html) transitions).
- Atomic transfer and single ownership
  - Ensures that only one workflow is managed (aka owned) by a single owner at a time in an atomic manner (including when the workflow is transferred to a owner that is resuming some other failed owners work). This avoids contention and ensures a workflow is managed by one and only one entity at a time.
  - *Note:* this does not mean that the owner needs to run the workflow itself but instead said owner could use an engine that runs the work in a distributed manner to ensure that the workflow progresses.
- Separation of workflow construction and execution
  - Jobs can be created with logbooks that contain a specification of the work to be done by a entity (such as an API server). The job then can be completed by a entity that is watching that jobboard (not necessarily the API server itself). This creates a disconnection between work formation and work completion that is useful for scaling out horizontally.
- Asynchronous completion
  - When for example a API server posts a job for completion to a jobboard that API server can return a *tracking* identifier to the user calling the API service. This *tracking* identifier can be used by the user to poll for status (similar in concept to a shipping *tracking* identifier created by fedex or UPS).



- 高可用
  - 通过将部分完成任务或未开始任务转移给能够续做或重启的实体来保证工作流的正常完成。
- 转移操作的原子性与归属的独占性
  - 保证任意时刻，一个工作流只能以原子性方式归属于一个实体。
  - 注意：
- workflow构建与执行的解耦
  - workflow的构建与执行不一定由同一个实体完成。
- 异步执行
  - Job是异步执行的。

## Usage

All jobboards are mere classes that implement same interface, and of course it is possible to import them and create instances of them just like with any other class in Python. But the easier (and recommended) way for creating jobboards is by using the [`fetch()`](https://docs.openstack.org/taskflow/latest/user/jobs.html#taskflow.jobs.backends.fetch) function which uses entrypoints (internally using [stevedore](https://docs.openstack.org/stevedore/latest)) to fetch and configure your backend.

推荐通过`fetch()`函数创建Jobboard对象。

```python
from taskflow.persistence import backends as persistence_backends
from taskflow.jobs import backends as job_backends

...
persistence = persistence_backends.fetch({
    "connection': "mysql",
    "user": ...,
    "password": ...,
})
book = make_and_save_logbook(persistence)
board = job_backends.fetch('my-board', {
    "board": "zookeeper",
}, persistence=persistence)
job = board.post("my-first-job", book)
...
```

通过遍历方式消费Job。

```python
import time

from taskflow import exceptions as exc
from taskflow.persistence import backends as persistence_backends
from taskflow.jobs import backends as job_backends

...
my_name = 'worker-1'
coffee_break_time = 60
persistence = persistence_backends.fetch({
    "connection': "mysql",
    "user": ...,
    "password": ...,
})
board = job_backends.fetch('my-board', {
    "board": "zookeeper",
}, persistence=persistence)
while True:
    my_job = None
    for job in board.iterjobs(only_unclaimed=True):
        try:
            board.claim(job, my_name)
        except exc.UnclaimableJob:
            pass
        else:
            my_job = job
            break
    if my_job is not None:
        try:
            perform_job(my_job)
        except Exception:
            LOG.exception("I failed performing job: %s", my_job)
            board.abandon(my_job, my_name)
        else:
            # I finished it, now cleanup.
            board.consume(my_job)
            persistence.get_connection().destroy_logbook(my_job.book.uuid)
    time.sleep(coffee_break_time)
...
```

通过`store`为Flow提供参数。

```python
from oslo_utils import uuidutils

from taskflow import engines
from taskflow.persistence import backends as persistence_backends
from taskflow.persistence import models
from taskflow.jobs import backends as job_backends


...
persistence = persistence_backends.fetch({
    "connection': "mysql",
    "user": ...,
    "password": ...,
})
board = job_backends.fetch('my-board', {
    "board": "zookeeper",
}, persistence=persistence)

book = models.LogBook('my-book', uuidutils.generate_uuid())

flow_detail = models.FlowDetail('my-job', uuidutils.generate_uuid())
book.add(flow_detail)

connection = persistence.get_connection()
connection.save_logbook(book)

flow_detail.meta['store'] = {'a': 1, 'c': 3}

job_details = {
    "flow_uuid": flow_detail.uuid,
    "store": {'a': 2, 'b': 1}
}

engines.save_factory_details(flow_detail, flow_factory,
                             factory_args=[],
                             factory_kwargs={},
                             backend=persistence)

jobboard = get_jobboard(zk_client)
jobboard.connect()
job = jobboard.post('my-job', book=book, details=job_details)

# the flow global parameters are now the combined store values
# {'a': 2, 'b': 1', 'c': 3}
...
```

## Hierarchy

![Inheritance diagram of taskflow.jobs.base, taskflow.jobs.backends.impl_redis, taskflow.jobs.backends.impl_zookeeper](https://docs.openstack.org/taskflow/latest/_images/inheritance-e896f0e266651aa414dca3d7d8cc50224c0edad6.png)