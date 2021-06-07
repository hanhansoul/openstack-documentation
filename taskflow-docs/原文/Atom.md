## Atom

An [`atom`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom) is the smallest unit in TaskFlow which acts as the base for other classes. Atoms have a name and may have a version. An atom is expected to name desired input values (requirements) and name outputs (provided values).

```
class taskflow.atom.Atom(name=None, 
                        provides=None, 
                        requires=None, 
                        auto_extract=True, 
                        rebind=None, 
                        inject=None, 
                        ignore_list=None, 
                        revert_rebind=None, 
                        revert_requires=None)
```

An atom is a named object that operates with input data to perform some action that furthers the overall flows progress. It usually also produces some of its own named output as a result of this process.

**Parameters**

- **name** – Meaningful name for this atom, should be something that is distinguishable and understandable for notification, debugging, storing and any other similar purposes.
- **provides** – A set, string or list of items that this will be providing (or could provide) to others, used to correlate and associate the thing/s this atom produces, if it produces anything at all.
- **inject** – An *immutable* input_name => value dictionary which specifies any initial inputs that should be automatically injected into the atoms scope before the atom execution commences (this allows for providing atom *local*values that do not need to be provided by other atoms/dependents).
- **rebind** – A dict of key/value pairs used to define argument name conversions for inputs to this atom’s `execute`method.
- **revert_rebind** – The same as `rebind` but for the `revert`method. If unpassed, `rebind` will be used instead.
- **requires** – A set or list of required inputs for this atom’s`execute` method.
- **revert_requires** – A set or list of required inputs for this atom’s `revert` method. If unpassed, `requires` will be used.

**Variables**

- **version** – An *immutable* version that associates version information with this atom. It can be useful in resuming older versions of atoms. Standard major, minor versioning concepts should apply.
- **save_as** – An *immutable* output `resource` name `OrderedDict` this atom produces that other atoms may depend on this atom providing. The format is output index (or key when a dictionary is returned from the execute method) to stored argument name.
- **rebind** – An *immutable* input `resource` `OrderedDict` that can be used to alter the inputs given to this atom. It is typically used for mapping a prior atoms output into the names that this atom expects (in a way this is like remapping a namespace of another atom into the namespace of this atom).
- **revert_rebind** – The same as `rebind` but for the revert method. This should only differ from `rebind` if the `revert`method has a different signature from `execute` or a different `revert_rebind` value was received.
- [**inject**](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.storage.Storage.inject) – See parameter `inject`.
- **Atom.name** – See parameter `name`.
- **optional** – A [`OrderedSet`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.sets.OrderedSet) of inputs that are optional for this atom to `execute`.
- **revert_optional** – The `revert` version of `optional`.
- [**provides**](https://docs.openstack.org/taskflow/latest/user/patterns.html#taskflow.flow.Flow.provides) – A [`OrderedSet`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.sets.OrderedSet) of outputs this atom produces.



**priority**

A numeric priority that instances of this class will have when running, used when there are multiple *parallel* candidates to execute and/or revert. During this situation the candidate list will be stably sorted based on this priority attribute which will result in atoms with higher priorities executing (or reverting) before atoms with lower priorities (higher being defined as a number bigger, or greater than an atom with a lower priority number). By default all atoms have the same priority (zero).

**requires**

A [`OrderedSet`](https://docs.openstack.org/taskflow/latest/user/types.html#taskflow.types.sets.OrderedSet) of inputs this atom requires to function.

**pre_execute()**

Code to be run prior to executing the atom.

A common pattern for initializing the state of the system prior to running atoms is to define some code in a base class that all your atoms inherit from.

**post_execute()**

Code to be run after executing the atom.

A common pattern for cleaning up global state of the system after the execution of atoms is to define some code in a base class that all your atoms inherit from.

**execute()**

Activate a given atom which will perform some operation and return.

**pre_revert()**

Code to be run prior to reverting the atom.

**post_revert()**

Code to be run after reverting the atom.

**revert()**

Revert this atom.

This method should undo any side-effects caused by previous execution of the atom using the result of the [`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.atom.Atom.execute) method and information on the failure which triggered reversion of the flow the atom is contained in (if applicable).



## Task

A [`task`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.task.Task) (derived from an atom) is a unit of work that can have an execute & rollback sequence associated with it (they are *nearly* analogous to functions). Your task objects should all derive from [`Task`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.task.Task) which defines what a task must provide in terms of properties and methods.

![Task outline.](https://docs.openstack.org/taskflow/latest/_images/tasks.png)

Currently the following *provided* types of task subclasses are:

- [`Task`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.task.Task): useful for inheriting from and creating your own subclasses.
- [`FunctorTask`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.task.FunctorTask): useful for wrapping existing functions into task objects.



## Retry

A [`retry`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry) (derived from an atom) is a special unit of work that handles errors, controls flow execution and can (for example) retry other atoms with other parameters if needed. The goal is that with this consultation the retry atom will suggest a *strategy* for getting around the failure (perhaps by retrying, reverting a single atom, or reverting everything contained in the retries associated [scope](http://en.wikipedia.org/wiki/Scope_(computer_science))).

Currently derivatives of the [`retry`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry) base class must provide a [`on_failure()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.on_failure)method to determine how a failure should be handled. The current enumeration(s) that can be returned from the [`on_failure()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.on_failure) method are defined in an enumeration class described here:

1. `REVERT` *= 'REVERT'*：Reverts only the surrounding/associated subflow.
2. `REVERT_ALL` *= 'REVERT_ALL'*：Reverts the entire flow, regardless of parent strategy.
3. `RETRY` *= 'RETRY'*：Retries the surrounding/associated subflow again.

To aid in the reconciliation process the [`retry`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry) base class also mandates[`execute()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.execute) and [`revert()`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Retry.revert) methods (although subclasses are allowed to define these methods as no-ops) that can be used by a retry atom to interact with the runtime execution model (for example, to track the number of times it has been called which is useful for the [`ForEach`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.ForEach) retry subclass).

To avoid recreating common retry patterns the following provided retry subclasses are provided:

- [`AlwaysRevert`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.AlwaysRevert): Always reverts subflow.
- [`AlwaysRevertAll`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.AlwaysRevertAll): Always reverts the whole flow.
- [`Times`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.Times): Retries subflow given number of times.
- [`ForEach`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.ForEach): Allows for providing different values to subflow atoms each time a failure occurs (making it possibly to resolve the failure by altering subflow atoms inputs).
- [`ParameterizedForEach`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.ParameterizedForEach): Same as [`ForEach`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.ForEach) but extracts values from storage instead of the [`ForEach`](https://docs.openstack.org/taskflow/latest/user/atoms.html#taskflow.retry.ForEach) constructor.



### Area of influence

Each retry atom is associated with a flow and it can *influence* how the atoms (or nested flows) contained in that flow retry or revert (using the previously mentioned patterns and decision enumerations):

![Retry area of influence](https://docs.openstack.org/taskflow/latest/_images/area_of_influence.svg)

In this diagram retry controller (1) will be consulted if task `A`, `B` or `C` fail and retry controller (2) decides to delegate its retry decision to retry controller (1). If retry controller (2) does **not** decide to delegate its retry decision to retry controller (1) then retry controller (1) will be oblivious of any decisions. If any of task `1`, `2` or `3` fail then only retry controller (1) will be consulted to determine the strategy/pattern to apply to resolve there associated failure.

```
class EchoTask(task.Task):
	def execute(self, *args, **kwargs):
		print(self.name)
        print(args)
        print(kwargs)

flow = linear_flow.Flow('f1').add(EchoTask('t1'),
		linear_flow.Flow('f2', retry=retry.ForEach(values=['a', 'b', 'c'], name='r1',      		provides='value'))
		.add(EchoTask('t2'),
		     EchoTask('t3', requires='value')),
		     EchoTask('t4'))
```



```
class SendMessage(task.Task):
	def execute(self, message):
		print("Sending message: %s" % message)

flow = linear_flow.Flow('send_message', retry=retry.Times(5)).add(SendMessage('sender'))
```



```
class ConnectToServer(task.Task):
	def execute(self, ip):
		print("Connecting to %s" % ip)

server_ips = ['192.168.1.1', '192.168.1.2', '192.168.1.3' ]
flow = linear_flow.Flow('send_message', 
	retry=retry.ParameterizedForEach(rebind={'values': 'server_ips'},provides='ip'))
	.add(ConnectToServer(requires=['ip']))
```



## Hierarchy

![Inheritance diagram of taskflow.atom, taskflow.task, taskflow.retry.Retry, taskflow.retry.AlwaysRevert, taskflow.retry.AlwaysRevertAll, taskflow.retry.Times, taskflow.retry.ForEach, taskflow.retry.ParameterizedForEach](https://docs.openstack.org/taskflow/latest/_images/inheritance-132045c46dd04436997cbb2633233e2fad13b700.png)