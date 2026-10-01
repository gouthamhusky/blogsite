---
title: "TIL: a Terminating namespace is a stuck finalizer"
date: 2026-10-01
---

A namespace sitting in `Terminating` for hours isn't the API server being slow. It's stuck because something inside it still carries a finalizer, and the controller responsible for removing that finalizer is gone.

Finalizers are just strings in `metadata.finalizers`. When you delete an object, the API server stamps a `deletionTimestamp` on it instead of removing it, and only actually deletes it once every finalizer in the list has been removed by the controller that owns it.

Deleting a namespace fans that out: everything in the namespace gets a delete call, and anything with a finalizer parks in `Terminating` until the cleanup happens. If the controller that was supposed to do the cleanup was uninstalled, crashed, or never ran in that cluster — CRDs left behind by a removed operator are the classic case — the object waits forever, and the namespace waits with it.

You can see exactly what's blocking:

```bash
kubectl api-resources --verbs=list --namespaced -o name \
  | xargs -n 1 kubectl get --show-kind --ignore-not-found -n stuck-ns
```

and inspect an object's finalizers directly:

```bash
kubectl get myresource example -n stuck-ns -o jsonpath='{.metadata.finalizers}'
```

If the leftover is genuinely safe to drop, removing the finalizer lets the deletion proceed:

```bash
kubectl patch myresource example -n stuck-ns \
  --type=json -p='[{"op":"remove","path":"/metadata/finalizers"}]'
```

{{< note "gotcha" >}}
Removing a finalizer by hand skips whatever cleanup the controller was supposed to do — deleting cloud resources, revoking credentials, draining connections. Only do it when you know what the finalizer was protecting, or you're trading a stuck namespace for a leak somewhere you can't see.
{{< /note >}}
