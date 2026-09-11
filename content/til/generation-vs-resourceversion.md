---
title: "TIL: generation and resourceVersion answer different questions"
date: 2026-09-05
---
Easy to mash these together when you're staring at `kubectl get -o yaml`.
They both go up. They mean different things.

`metadata.resourceVersion` is an opaque etcd/API concurrency token. Any
write that changes the persisted object — spec, status, labels, finalizers,
annotations — can bump it. You use it for optimistic concurrency
(`resourceVersion` match on update) and for watches. Comparing two
`resourceVersion` strings as numbers is a bad habit; treat them as opaque.

`metadata.generation` is specifically about desired state. The API server
increments it when the *spec* (or other generation-tracked fields) change.
Pure status updates from a controller typically leave `generation` alone.
That's why controllers can say 'I reconciled generation N' without caring
about every noisy status write in between.

```yaml
metadata:
  generation: 4
  resourceVersion: "184291"
status:
  observedGeneration: 4
```

If you're deciding whether a controller has caught up to the *user's*
intent, look at `generation` / `observedGeneration`. If you're doing a
conditional update or debugging watch lag, look at `resourceVersion`.

{{< note "gotcha" >}}
Not every resource increments `generation` the way you expect — and some
status-only subresources mean the main object's `resourceVersion` still
moves while `generation` stays put. When in doubt, change the spec once
and watch which field actually ticks.
{{< /note >}}

Same cluster, two clocks: one for 'did the desired state change?', one for
'has this object byte sequence changed at all?'.
