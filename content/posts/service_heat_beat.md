---
author: jieteki
categories:
- 技术文章
- 云计算
date: 2017-01-15T12:06:42+08:00
description: ""
keywords:
- openstack
title: Service heat beat
url: ""
---

In a cluster with many services running, we need to know which service is currently up running, and which services are down. usually it is called `heart beat` checking.

<!--more-->

One approch is, each service periodically report its state to a controller node by sending a heat beat request, controller then know which service node is down when the service didn't send request for quite a long time.  
Another approch is, each service periodically write to a database table, controller node can check the table, and know which service is down, if the service didn't update the table for a long time.

For openstack cinder-volume service, it's using the second method.
Let's review the code, to find out how it is impletemented

In openstack environment, checking service status can be done by `xxxx service-list` cli command, for example `cinder service-list` lists every cinder-volume service in cluster; `nova service-list` lists every nova-compute service in cluster.

`cidner service-list` is handle by `cinder-api` wsgi web server, typically, the `ServiceController`

cinder/cinder/api/contrib/services.py

```python
class ServiceController(wsgi.Controller):
    ...
    @wsgi.serializers(xml=ServicesIndexTemplate)
    def index(self, req):
        ...
        svcs = []
        for svc in services:
            updated_at = svc.updated_at
            delta = now - (svc.updated_at or svc.created_at)
            delta_sec = delta.total_seconds()
            if svc.modified_at:
                delta_mod = now - svc.modified_at
                if abs(delta_sec) >= abs(delta_mod.total_seconds()):
                    updated_at = svc.modified_at
            alive = abs(delta_sec) <= CONF.service_down_time
            art = (alive and "up") or "down"
            active = 'enabled'
            if svc.disabled:
                active = 'disabled'
            if updated_at:
                updated_at = timeutils.normalize_time(updated_at)
            ret_fields = {'binary': svc.binary, 'host': svc.host,
                          'zone': svc.availability_zone,
                          'status': active, 'state': art,
                          'updated_at': updated_at}
            ...
            svcs.append(ret_fields)
```

it first fetch all services from database table, then check updated\_at or created\_at time from now. when `delta_sec` is greater than `CONF.service_down_time` (by default, it is 1 minute), then the service is considered to be dead.

So when does service update the table? check `Service` class in cinder module. work is done by `report_state` method, which update the model by increasing `report_count` field.

cinder/cinder/service.py

```python
class Service(service.Service):
    ...
    def report_state(self):
        try:
            try:
                service_ref = objects.Service.get_by_id(ctxt, self.service_id)
            except exception.NotFound:
                LOG.debug('The service database object disappeared, '
                          'recreating it.')
                self._create_service_ref(ctxt)
                service_ref = objects.Service.get_by_id(ctxt, self.service_id)

            service_ref.report_count += 1
            if zone != service_ref.availability_zone:
                service_ref.availability_zone = zone

            service_ref.save()
        ...
    ...
```

Actually, `report_state` is a looping call

cinder/cinder/service.py

```python
class Service(service.Service):
    ...
    def start(self):
        if self.report_interval:
            pulse = loopingcall.FixedIntervalLoopingCall(
                self.report_state)
            pulse.start(interval=self.report_interval,
                        initial_delay=self.report_interval)
            self.timers.append(pulse)
        ...
    ...
```

Let’s see how this `FixedIntervalLoopingCall` will start.

oslo.service/oslo\_service/loopingcall.py

```python
class LoopingCallBase(object):
    ...
    def _on_done(self, gt, *args, **kwargs):
        self._thread = None
        self._running = False

    def _start(self, idle_for, initial_delay=None, stop_on_exception=True):
        if self._thread is not None:
            raise RuntimeError(self._RUN_ONLY_ONE_MESSAGE)
        self._running = True
        self.done = event.Event()
        self._thread = greenthread.spawn(
            self._run_loop, idle_for,
            initial_delay=initial_delay, stop_on_exception=stop_on_exception)
        self._thread.link(self._on_done)
        return self.done
    ...
    def _run_loop(self, idle_for_func,
                  initial_delay=None, stop_on_exception=True):
        ...
        func = self.f if stop_on_exception else _safe_wrapper(self.f, kind,
                                                              func_name)
        try:
            watch = timeutils.StopWatch()
            while self._running:
                watch.restart()
                result = func(*self.args, **self.kw)
                watch.stop()
                if not self._running:
                    break
                idle = idle_for_func(result, watch.elapsed())
                LOG.trace('%(kind)s %(func_name)r sleeping '
                          'for %(idle).02f seconds',
                          {'func_name': func_name, 'idle': idle,
                           'kind': kind})
                greenthread.sleep(idle)
        except LoopingCallDone as e:
            self.done.send(e.retvalue)
        ...
        else:
            self.done.send(True)

class FixedIntervalLoopingCall(LoopingCallBase):
    ...
    def start(self, interval, initial_delay=None, stop_on_exception=True):
        def _idle_for(result, elapsed):
            delay = round(elapsed - interval, 2)
            if delay > 0:
                func_name = reflection.get_callable_name(self.f)
                LOG.warning(_LW('Function %(func_name)r run outlasted '
                                'interval by %(delay).2f sec'),
                            {'func_name': func_name, 'delay': delay})
            return -delay if delay < 0 else 0
        return self._start(_idle_for, initial_delay=initial_delay,
                           stop_on_exception=stop_on_exception)

```

The code is pretty straight: `LoopingCallBase` spawn a greenthread, which is running `_run_loop()`, this function has a while loop executing `self.f` again and again.
this while loop will end when `self.f` raise Exception, or someone calling `stop` api.

Sending `heart beat` request should be done in seperate thread, so that main thread heavy load job would not interrupt heat beat. Maybe the service is busy doing some job, but it is still alive.%
