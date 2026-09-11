---
title: "TIL: generateName lets the API server pick a unique name for you"
date: 2026-09-11
---
I always thought unique object names were a client problem — pick a UUID,
or fail and retry. Turns out the API server has a native escape hatch:
`metadata.generateName`.

On create, if you set `generateName` *instead of* `metadata.name`, the
API server treats your value as a prefix and appends a random suffix so
the resulting name is unique. You ask for `job-`; you get something like
`job-x7k2m`.

```yaml
apiVersion: v1
kind: Pod
metadata:
  generateName: debug-
spec:
  containers:
    - name: pause
      image: registry.k8s.io/pause:3.9
```

That's how Jobs, ReplicaSets, and a bunch of controllers mint Pod names
without coordinating a global counter. The usual generator adds five
alphanumeric characters and truncates a long prefix so the full name
stays within the 63-character limit.

{{< note "gotcha" >}}
`generateName` only applies when `name` is unset. If both are present,
`name` wins. And uniqueness still isn't absolute — a collision can return
HTTP 409. From Kubernetes v1.31 the server retries generation (up to
eight attempts) before giving up, so 409s got rarer, not impossible.
{{< /note >}}

So when you need 'a name like this, but unique,' you don't have to invent
one yourself. Hand the API server a prefix and let it finish the string.
