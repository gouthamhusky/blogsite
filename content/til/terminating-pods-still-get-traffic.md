---
title: "TIL: a Terminating pod can still get traffic for a bit"
date: 2026-09-08
---
I assumed once a pod showed `Terminating`, Services had already dropped
it. Not quite — there's a window where the container is winding down and
the load balancer / kube-proxy path may still be sending it requests.

On delete, the pod gets a `deletionTimestamp` and kubelet starts the
graceful shutdown (`terminationGracePeriodSeconds`, preStop hooks, then
SIGTERM). Endpoint / EndpointSlice removal is racing that shutdown, not
guaranteeing 'zero traffic before SIGTERM'. If your app keeps accepting
connections until it dies, those late requests fail or get cut mid-flight.

The usual fix is to fail readiness as soon as shutdown starts — or use a
`preStop` sleep / hook so endpoint propagation can finish before the
process stops listening:

```yaml
spec:
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
  terminationGracePeriodSeconds: 30
```

Readiness going `False` is what pulls you out of Service endpoints.
`Terminating` alone is not a traffic barrier.

{{< note "gotcha" >}}
`sleep` in `preStop` only helps if `terminationGracePeriodSeconds` is long
enough to cover the sleep *plus* your actual shutdown. Burn the whole
grace period on sleep and kubelet will SIGKILL you mid-cleanup.
{{< /note >}}

So 'pod is Terminating' means the control plane started deletion — not
that the mesh, the NLB, or kube-proxy already agreed to leave it alone.
