---
title: "TIL: a finalizer doesn't mean 'important' — it means 'don't delete me yet'"
date: 2026-09-02
---
I used to treat finalizers as a vague 'this object matters' flag. They're
not. They're a deletion lock: the API server will mark the object for
deletion, but it won't actually remove it from etcd until every finalizer
is gone.

When you `kubectl delete` something with finalizers, what you usually see
is `metadata.deletionTimestamp` get set. From that point the object is
terminating. Controllers watching for that timestamp are supposed to do
their cleanup, then patch the object to remove *their* finalizer from
`metadata.finalizers`. Only when the list is empty does the API server
finish the delete.

That's why a Namespace or a custom resource can sit in `Terminating`
forever — something still has a finalizer registered and nobody cleared
it (controller crashed, CRD gone, webhook dead, operator uninstalled
wrong).

```yaml
metadata:
  name: stuck-thing
  deletionTimestamp: "2026-09-02T12:00:00Z"
  finalizers:
    - example.com/cleanup
```

{{< note "gotcha" >}}
Force-removing a finalizer with `kubectl edit` / a patch *will* let the
object disappear — and it skips whatever cleanup that finalizer was
guarding. Fine for a lab stuck Namespace. Less fine when the finalizer
was holding cloud resources, volume cleanup, or ledger state.
{{< /note >}}

So when something won't die, don't start with 'Kubernetes is broken'.
Look at `deletionTimestamp` and `finalizers` first. The gap between
'I asked to delete it' and 'it's gone' is exactly those strings.
