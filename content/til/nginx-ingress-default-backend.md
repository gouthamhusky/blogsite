---
title: "TIL: the ingress-nginx 404 page is just another Service"
date: 2026-08-24
---
I'd always assumed a 404 from ingress-nginx was nginx doing something
built-in and untouchable. It isn't — it's routing you to an actual
Kubernetes Service, and you can point it at your own.

Every Ingress rule you write matches on a host and a set of paths. If a
request comes in that doesn't match any rule on any Ingress in the
cluster, ingress-nginx doesn't invent a response — it forwards the
request to whatever Service is configured as its **default backend**,
and that Service's response is what the client sees. Out of the box,
that's the small `defaultbackend-amd64` pod the chart installs for you,
which does nothing but serve a plain 404.

Because it's just a Service, you can replace it with your own — a
branded 404 page, a catch-all that logs unmatched requests, whatever you
need. Point the controller at it with a command-line flag:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  template:
    spec:
      containers:
        - name: controller
          args:
            - /nginx-ingress-controller
            - --default-backend-service=my-namespace/my-custom-404
```

{{< note "gotcha" >}}
The flag takes `namespace/service-name`, not just a service name — and
it's read at controller startup, not reloaded live. Changing it means
restarting the controller pods, same as any other startup flag.
{{< /note >}}

Nothing about how Ingress matching works changes. It just means the
fallback isn't a hardcoded nginx behavior — it's a Service like any
other, sitting there waiting for whatever nobody else claimed.
