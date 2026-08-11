---
title: "TIL: a Service doesn't know what a Deployment is"
date: 2026-08-10
---
I'd always assumed a Service was wired to a specific Deployment. It isn't
— a Service only matches on pod labels. It has no concept of Deployment or
ReplicaSet at all, which means nothing stops two different Deployments
from feeding the same Service.

That's enough to fake a canary rollout without any extra tooling. Give
your canary Deployment a single replica, your stable Deployment nine, and
make sure both pod templates carry the label your Service selects on:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        track: stable
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        track: canary
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
```

The Service's `spec.selector` only cares about `app: web` — it can't tell
`web-stable` and `web-canary` apart, so it load-balances across all ten
matching pods regardless of which Deployment owns them.

{{< note "gotcha" >}}
This gets you a *replica-ratio* split, not a precise one — 1 pod out of 10
is roughly 10% of requests, subject to kube-proxy and any session
affinity, not an exact weighted percentage. If you need real weighted
traffic shifting, that's what tools like Argo Rollouts or Flagger are for.
{{< /note >}}

No admission webhook, no service mesh, no extra CRDs — just two
Deployments agreeing to share a label.
