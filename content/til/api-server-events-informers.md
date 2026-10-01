---
title: "TIL: informers just subscribe to the API server's event stream"
date: 2026-09-30
---

Every create, update, and delete that hits the API server becomes a watch event on that object's resource version stream. The same stream `kubectl get pods -w` listens to — `ADDED` / `MODIFIED` / `DELETED`, delivered over a long-lived watch connection, no polling anywhere.

The informer pattern in client-go is what turns that stream into something usable. It keeps a local, thread-safe **store** in sync with the watch, and your event handlers mostly just push keys onto a workqueue:

```go
factory := informers.NewSharedInformerFactory(clientset, 0)
podInformer := factory.Core().V1().Pods().Informer()

podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { /* enqueue key */ },
    UpdateFunc: func(old, new interface{}) { /* enqueue key */ },
    DeleteFunc: func(obj interface{}) { /* enqueue key */ },
})

go factory.Start(ctx.Done())
factory.WaitForCacheSync(ctx.Done())
```

The reconcile then reads from the local cache instead of hammering the API server on every event.

{{< note "gotcha" >}}
Watch connections break, and that's expected — the informer re-lists and resumes from the last resource version it saw. But the re-list means handlers can see **duplicate `MODIFIED` events** for objects that never really changed, so reconcile logic should always be level-triggered (converge current state to desired state) rather than assuming every event is a real change.

Same reason `UpdateFunc` fires on resyncs and status writes, not just spec changes — filter in the handler if you only care about specific fields.
{{< /note >}}

So "watching" in Kubernetes isn't a metaphor. There is a real event stream coming off the API server, and informers are subscribers with a cache attached.
