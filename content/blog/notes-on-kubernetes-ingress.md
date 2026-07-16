---
title: "some notes on kubernetes ingress"
date: 2025-07-14
categories: ["kubernetes"]
tags: ["networking"]
---
I keep re-learning how ingress actually routes traffic, so here are the notes I wish I'd written the first time.

## the mental model

An Ingress object is just *configuration*. On its own it does nothing — you need an ingress **controller** running in the cluster to read those objects and actually route traffic.

{{< note "gotcha" >}}
If your Ingress resource exists but nothing happens, 90% of the time you forgot to install a controller, or installed one that isn't watching your ingress class.
{{< /note >}}

## a minimal example

Here's the smallest thing that does something useful:

{{< file "ingress.yaml" >}}
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: nginx
  rules:
    - host: example.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```
{{< /file >}}

The `ingressClassName` is the part people miss. Without it, a controller that's scoped to a class will quietly ignore your resource.

## checking what's happening

When it doesn't work, start here:

```bash
kubectl get ingress
kubectl describe ingress web
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
```

That last one — reading the controller logs — is almost always what actually tells you the answer.
