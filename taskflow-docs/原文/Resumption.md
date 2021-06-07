# Resumption

## Overview

**Question**: *How can we persist the flow so that it can be resumed, restarted or rolled-back on engine failure?*

**Answer:** Since a flow is a set of [atoms](https://docs.openstack.org/taskflow/latest/user/atoms.html) and relations between atoms we need to create a model and corresponding information that allows us to persist the *right* amount of information to preserve, resume, and rollback a flow on software or hardware failure.

To allow for resumption TaskFlow must be able to re-create the flow and re-connect the links between atom (and between atoms->atom details and so on) in order to revert those atoms or resume those atoms in the correct ordering. TaskFlow provides a pattern that can help in automating this process (it does **not** prohibit the user from creating their own strategies for doing this).

为了实现Flow的执行在因为Engine出现异常中断后能够续做、重启或回滚，TaskFlow需要支持Flow运行相关信息的持久化功能，用于在Engine重启后能够还原Flow运行中断时的场景。

## Factories

<u>The default provided way is to provide a [factory](https://en.wikipedia.org/wiki/Factory_(object-oriented_programming)) function which will create (or recreate your workflow).</u> <u>This function can be provided when loading a flow and corresponding engine via the provided [`load_from_factory()`](https://docs.openstack.org/taskflow/latest/user/engines.html#taskflow.engines.helpers.load_from_factory) method.</u> This [factory](https://en.wikipedia.org/wiki/Factory_(object-oriented_programming)) function is expected to be a function (or `staticmethod`) which is reimportable (aka has a well defined name that can be located by the `__import__` function in python, this excludes `lambda` style functions and `instance` methods). <u>The [factory](https://en.wikipedia.org/wiki/Factory_(object-oriented_programming)) function name will be saved into the logbook and it will be imported and called to create the workflow objects</u> (or recreate it if resumption happens). This allows for the flow to be recreated if and when that is needed (even on remote machines, as long as the reimportable name can be located).

## Names

When a flow is created it is expected that each atom has a unique name, this name serves a special purpose in the resumption process (as well as serving a useful purpose when running, allowing for atom identification in the [notification](https://docs.openstack.org/taskflow/latest/user/notifications.html) process). The reason for having names is that an atom in a flow needs to be somehow matched with (a potentially) existing [`AtomDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.AtomDetail)during engine resumption & subsequent running.

The match should be:

- stable if atoms are added or removed
- should not change when service is restarted, upgraded…
- should be the same across all server instances in HA setups

Names provide this although they do have weaknesses:

- the names of atoms must be unique in flow
- it becomes hard to change the name of atom since a name change causes other side-effects

Flow中的Atom都应该给定一个唯一标识名，该表示名用于在Engine重启时与持久化的AtomDetail对象进行匹配。

## Scenarios

When new flow is loaded into engine, there is no persisted data for it yet, so a corresponding [`FlowDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.FlowDetail) object will be created, as well as a [`AtomDetail`](https://docs.openstack.org/taskflow/latest/user/persistence.html#taskflow.persistence.models.AtomDetail) object for each atom that is contained in it. These will be immediately saved into the persistence backend that is configured. If no persistence backend is configured, then as expected nothing will be saved and the atoms and flow will be ran in a non-persistent manner.

当一个新Flow装载到Engine中时，此时没有持久化的Flow信息存在，因此会创建一个新的FlowDetail对象，同时为Flow中的每一个Atom创建一个AtomDetail对象，并将所有对象持久化到Backend中。

**Subsequent run:** When we resume the flow from a persistent backend (for example, if the flow was interrupted and engine destroyed to save resources or if the service was restarted), we need to re-create the flow. For that, we will call the function that was saved on first-time loading that builds the flow for us (aka; the flow factory function described above) and the engine will run. The following scenarios explain some expected structural changes and how they can be accommodated (and what the effect will be when resuming & running).

当从持久化Backend中重启一个Flow时，我们需要重新构建这个Flow。

### Same atoms

### Atom was added

### Atom was removed

### Atom code was changed

### Atom was split in two atoms or merged

### Flow structure was changed