---
author: jieteki
categories:
- 技术文章
- 云计算
date: 2017-01-15T15:06:42+08:00
description: ""
keywords:
- openstack
title: Periodic task in cinder
url: ""
---


In openstack, there are a lot of periodically running jobs, like report state, report driver status, report capabilities etc.
The component manager are often responsible for these kind of jobs. like `VolumeManager`.

<!--more-->

When we take a close look at these manager's code, we can see some methods are decorated by  `@periodic_task.periodic_task`,
for example:

```python
from oslo_service import periodic_task

class VolumeManager(manager.SchedulerDependentManager):
    ...
    @periodic_task.periodic_task
    def _report_driver_status(self, context):
        ...
```

From the decorator name `periodic_task`, we know these kinds of methods are meaned to be running periodicaly, so how are they implemented?

file: oslo.service/oslo_service/periodic_task.py

```python
def periodic_task(*args, **kwargs):
    """Decorator to indicate that a method is a periodic task.

    This decorator can be used in two ways:

        1. Without arguments '@periodic_task', this will be run on the default
           interval of 60 seconds.

        2. With arguments:
           @periodic_task(spacing=N [, run_immediately=[True|False]]
           [, name=[None|"string"])
           this will be run on approximately every N seconds. If this number is
           negative the periodic task will be disabled. If the run_immediately
           argument is provided and has a value of 'True', the first run of the
           task will be shortly after task scheduler starts.  If
           run_immediately is omitted or set to 'False', the first time the
           task runs will be approximately N seconds after the task scheduler
           starts. If name is not provided, __name__ of function is used.
    """
    def decorator(f):
        # Test for old style invocation
        if 'ticks_between_runs' in kwargs:
            raise InvalidPeriodicTaskArg(arg='ticks_between_runs')

        # Control if run at all
        f._periodic_task = True
        f._periodic_external_ok = kwargs.pop('external_process_ok', False)
        f._periodic_enabled = kwargs.pop('enabled', True)
        f._periodic_name = kwargs.pop('name', f.__name__)

        # Control frequency
        f._periodic_spacing = kwargs.pop('spacing', 0)
        f._periodic_immediate = kwargs.pop('run_immediately', False)
        if f._periodic_immediate:
            f._periodic_last_run = None
        else:
            f._periodic_last_run = now()
        return f

    if kwargs:
        return decorator
    else:
        return decorator(args[0])
```

This decorator is a little different. Normally we use a decorator to do something before and after a method call, like below:

```python
import functools

def footprint(f):
    """Decorator to log info before and after method call.
    """
    @functools.wraps(f)
    def decorator(*args, **kwargs):
        logger.info('before call')
        return f(*args, **kwargs)
        logger.info('after call')

    return decorator
```

but the `periodic_task` decorator is different: all it does is to attach some `_periodic_****` attribute to the function. SO who will consume these attributes?

But first lets see VolumeManager inheritance tree.

```python
# file: oslo.service/oslo_service/periodic_task.py
class _PeriodicTasksMeta(type):
    ...

@six.add_metaclass(_PeriodicTasksMeta)
class PeriodicTasks(object):
    ...


# file: cinder/cinder/manager.py
class PeriodicTasks(periodic_task.PeriodicTasks):
    ...

class Manager(base.Base, PeriodicTasks):
    ...

class SchedulerDependentManager(Manager):
    ...


# file: cinder/cinder/volume/manager.py
class VolumeManager(manager.SchedulerDependentManager):
    ...
    @periodic_task.periodic_task
    def _report_driver_status(self, context):
        ...
```

NOTE: `_PeriodicTasksMeta` and `PeriodicTasks` are in `periodic_task` module.

The magic is coming from `_PeriodicTasksMeta`. It's a metaclass, metaclass can define the actual class `PeriodicTasks`'s behaviour.

file: oslo.service/oslo\_service/periodic\_task.py

```python
class _PeriodicTasksMeta(type):
    def _add_periodic_task(cls, task):
        """Add a periodic task to the list of periodic tasks.

        The task should already be decorated by @periodic_task.

        :return: whether task was actually enabled
        """
        name = task._periodic_name
        ...
        cls._periodic_tasks.append((name, task))
        cls._periodic_spacing[name] = task._periodic_spacing
        return True

    def __init__(cls, names, bases, dict_):
        """Metaclass that allows us to collect decorated periodic tasks."""
        super(_PeriodicTasksMeta, cls).__init__(names, bases, dict_)

        # NOTE(sirp): if the attribute is not present then we must be the base
        # class, so, go ahead an initialize it. If the attribute is present,
        # then we're a subclass so make a copy of it so we don't step on our
        # parent's toes.
        try:
            cls._periodic_tasks = cls._periodic_tasks[:]
        except AttributeError:
            cls._periodic_tasks = []

        try:
            cls._periodic_spacing = cls._periodic_spacing.copy()
        except AttributeError:
            cls._periodic_spacing = {}

        for value in cls.__dict__.values():
            if getattr(value, '_periodic_task', False):
                cls._add_periodic_task(value)

```

The `__init__` method says:

when your class (`VolumeManager`) is created, find out all its methods, collect those method who has attribute `_periodic_task` and add the method to `cls._periodic_tasks`.

Then those decorated method are saved to `_periodic_tasks` array. So who will consume this array?

Let’s take a look of Service `start`.

file: cinder/cinder/service.py

```python
class Service(service.Service):
    ...
    def start(self):
        ...
        if self.periodic_interval:
            ...
            periodic = loopingcall.FixedIntervalLoopingCall(
                self.periodic_tasks)
            periodic.start(interval=self.periodic_interval,
                           initial_delay=initial_delay)
            self.timers.append(periodic)
    ...
    def periodic_tasks(self, raise_on_error=False):
        ...
        self.manager.periodic_tasks(ctxt, raise_on_error=raise_on_error)
    ...
```

It’s a LoopingCall, every `interval` seconds, it will call `periodic_tasks`, which is actually call `manager.periodic_tasks`

file: cinder/cinder/manager.py

```python
class Manager(base.Base, PeriodicTasks):
    ...
    def periodic_tasks(self, context, raise_on_error=False):
        """Tasks to be run at a periodic interval."""
        return self.run_periodic_tasks(context, raise_on_error=raise_on_error)
    ...
```

file: oslo.service/oslo_service/periodic_task.py

```python
@six.add_metaclass(_PeriodicTasksMeta)
class PeriodicTasks(object):
    ...
    def run_periodic_tasks(self, context, raise_on_error=False):
        ...
        for task_name, task in self._periodic_tasks:
            ...
            try:
                task(self, context)
            except Exception:
                if raise_on_error:
                    raise
            time.sleep(0)
    ...
```

Things are clear: Tasks are collected and saved in an array at Manager class when the class is created, a LoopingCall periodicaly fetch tasks from the array, running all these tasks every `interval` time.









