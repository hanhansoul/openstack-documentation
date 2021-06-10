# Engine

## 创建

```python
taskflow.engines.helpers.load(flow, 
                              store=None, 
                              flow_detail=None, 
                              book=None, 
                              backend=None, 
                              namespace='taskflow.engines', 
                              engine='default', 
                              **kwargs)

```

- **flow** – flow to load
- **store** – dict – data to put to storage to satisfy flow requirements
- **flow_detail** – FlowDetail that holds the state of the flow (if one is not provided then one will be created for you in the provided backend)
- **book** – LogBook to create flow detail in if flow_detail is None
- **backend** – storage backend to use or configuration that defines it
- **namespace** – driver namespace for stevedore (or empty for default)
- **engine** – string engine type or URI string with scheme that contains the engine type and any URI specific components that will become part of the engine options.
- **kwargs** – arbitrary keyword arguments passed as options (merged with any extracted `engine`), typically used for any engine specific options that do not fit as any of the existing arguments.



```python
taskflow.engines.helpers.run(flow, 
                             store=None, 
                             flow_detail=None, 
                             book=None, 
                             backend=None, 
                             namespace='taskflow.engines', 
                             engine='default', 
                             **kwargs)

```



```python
taskflow.engines.helpers.save_factory_details(flow_detail, 
                                              flow_factory, 
                                              factory_args, 
                                              factory_kwargs, 
                                              backend=None)
```

- **flow_detail** – FlowDetail that holds state of the flow to load
- **flow_factory** – function or string: function that creates the flow
- **factory_args** – list or tuple of factory positional arguments
- **factory_kwargs** – dict of factory keyword arguments
- **backend** – storage backend to use or configuration



```python
taskflow.engines.helpers.load_from_factory(flow_factory, 
                                           factory_args=None, 
                                           factory_kwargs=None, 
                                           store=None, 
                                           book=None, 
                                           backend=None, 
                                           amespace='taskflow.engines', 
                                           engine='default', 
                                           **kwargs)
```

- **flow_factory** – function or string: function that creates the flow
- **factory_args** – list or tuple of factory positional arguments
- **factory_kwargs** – dict of factory keyword arguments



```python
taskflow.engines.helpers.flow_from_detail(flow_detail)
```

- **flow_detail** – FlowDetail that holds state of the flow to load



```python
taskflow.engines.helpers.load_from_detail(flow_detail, 
                                          store=None, 
                                          backend=None, 
                                          namespace='taskflow.engines', 
                                          engine='default', 
                                          **kwargs)
```

- **flow_detail** – FlowDetail that holds state of the flow to load



## 类型



### 接口与实现

```python
class taskflow.engines.base.Engine(flow, flow_detail, backend, options)


class taskflow.engines.action_engine.engine.ActionEngine(flow, 
                                                         flow_detail, 
                                                         backend, 
                                                         options)

class taskflow.engines.action_engine.engine.SerialActionEngine(flow, 
                                                               flow_detail, 
                                                               backend, 
                                                               options)

class taskflow.engines.action_engine.engine.ParallelActionEngine(flow, 
                                                                 flow_detail, 
                                                                 backend, 
                                                                 options)

```

