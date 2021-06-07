## Terminology

1. Client: Code or program or service (or user) that uses this library to define flows and run them via engines.
2. Transport + protocol: Mechanism (and protocol on top of that mechanism) used to pass information between the client and worker (for example amqp as a transport and a json encoded message format as the protocol).
3. Executor: Part of the worker-based engine and is used to publish task requests, so these requests can be accepted and processed by remote workers.
4. Worker: Workers are started on remote hosts and each has a list of tasks it can perform (on request). Workers accept and process task requests that are published by an executor. Several requests can be processed simultaneously in separate threads (or processes…). For example, an executor can be passed to the worker and configured to run in as many threads (green or not) as desired.



## High level architecture

![../_images/worker-engine.svg](https://docs.openstack.org/taskflow/latest/_images/worker-engine.svg)



## Executor and worker communication

Let’s consider how communication between an executor and a worker happens.

1. The executor initiates task execution/reversion using a proxy object.
2. Proxy publishes task request (format is described below) into a named exchange using a routing key that is used to deliver request to particular workers topic. The executor then waits for the task requests to be accepted and confirmed by workers. If the executor doesn’t get a task confirmation from workers within the given timeout the task is considered as timed-out and a timeout exception is raised.
3. A worker receives a request message and starts a new thread for processing it.
   1. The worker dispatches the request (gets desired endpoint that actually executes the task).
   2. If dispatched succeeded then the worker sends a confirmation response to the executor otherwise the worker sends a failed response along with a serialized [`failure`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.failure.Failure) object that contains what has failed (and why).
   3. The worker executes the task and once it is finished sends the result back to the originating executor (every time a task progress event is triggered it sends progress notification to the executor where it is handled by the engine, dispatching to listeners and so-on).
   4. The executor gets the task request confirmation from the worker and the task request state changes from the `PENDING` to the `RUNNING` state. Once a task request is in the `RUNNING` state it can’t be timed-out (considering that the task execution process may take an unpredictable amount of time).
   5. The executor gets the task execution result from the worker and passes it back to the executor and worker-based engine to finish task processing (this repeats for subsequent tasks).



## Request state transitions

![WBE request state transitions](https://docs.openstack.org/taskflow/latest/_images/wbe_request_states.svg)

1. **WAITING** - Request placed on queue (or other [kombu](http://kombu.readthedocs.org/) message bus/transport) but not *yet* consumed.

2. **PENDING** - Worker accepted request and is pending to run using its executor (threads, processes, or other).
3. **FAILURE** - Worker failed after running request (due to task exception) or no worker moved/started executing (by placing the request into `RUNNING` state) with-in specified time span (this defaults to 60 seconds unless overridden).
4. **RUNNING** - Workers executor (using threads, processes…) has started to run requested task (once this state is transitioned to any request timeout no longer becomes applicable; since at this point it is unknown how long a task will run since it can not be determined if a task is just taking a long time or has failed).
5. **SUCCESS** - Worker finished running task without exception.



## Workers

To use the worker based engine a set of workers must first be established on remote machines. These workers must be provided a list of task objects, task names, modules names (or entrypoints that can be examined for valid tasks) they can respond to (this is done so that arbitrary code execution is not possible).

**Example:**

```python
from taskflow.engines.worker_based import worker as w

config = {
    'url': 'amqp://guest:guest@localhost:5672//',
    'exchange': 'test-exchange',
    'topic': 'test-tasks',
    'tasks': ['tasks:TestTask1', 'tasks:TestTask2'],
}
worker = w.Worker(**config)
worker.run()
```



## Engines

To use the worker based engine a flow must be constructed (which contains tasks that are visible on remote machines) and the specific worker based engine entrypoint must be selected. Certain configuration options must also be provided so that the transport backend can be configured and initialized correctly. Otherwise the usage should be mostly transparent (and is nearly identical to using any other engine type).

**Example with amqp transport:**

```python
flow = lf.Flow('simple-linear').add(...)
eng = taskflow.engines.load(flow, engine='worker-based',
                            url='amqp://guest:guest@localhost:5672//',
                            exchange='test-exchange',
                            topics=['topic1', 'topic2'])
eng.run()
```

**Example with filesystem transport:**

```python
flow = lf.Flow('simple-linear').add(...)
eng = taskflow.engines.load(flow, engine='worker-based',
                            exchange='test-exchange',
                            topics=['topic1', 'topic2'],
                            transport='filesystem',
                            transport_options={
                                'data_folder_in': '/tmp/in',
                                'data_folder_out': '/tmp/out',
                            })
eng.run()
```

Additional supported keyword arguments:

- `executor`: a class that provides a [`WorkerTaskExecutor`](https://docs.openstack.org/taskflow/latest/user/workers.html#taskflow.engines.worker_based.executor.WorkerTaskExecutor) interface; it will be used for executing, reverting and waiting for remote tasks.

- Atoms inside a flow must receive and accept parameters only from the ways defined in [persistence](https://docs.openstack.org/taskflow/latest/user/persistence.html). In other words, the task that is created when a workflow is constructed will not be the same task that is executed on a remote worker (and any internal state not passed via the [input and output](https://docs.openstack.org/taskflow/latest/user/inputs_and_outputs.html) mechanism can not be transferred). This means resource objects (database handles, file descriptors, sockets, …) can **not** be directly sent across to remote workers (instead the configuration that defines how to fetch/create these objects must be instead).
- Worker-based engines will in the future be able to run lightweight tasks locally to avoid transport overhead for very simple tasks (currently it will run even lightweight tasks remotely, which may be non-performant).
- Fault detection, currently when a worker acknowledges a task the engine will wait for the task result indefinitely (a task may take an indeterminate amount of time to finish). In the future there needs to be a way to limit the duration of a remote workers execution (and track their liveness) and possibly spawn the task on a secondary worker if a timeout is reached (aka the first worker has died or has stopped responding).