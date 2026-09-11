---
title: "TIL: a status condition is stale if observedGeneration doesn't match generation"
date: 2026-09-11
---
I used to treat `Ready=True` as current truth. It isn't — not if the
condition was written against an older generation of the resource.

Every time you change a Kubernetes object's `.spec`, the API server bumps
`metadata.generation`. Controllers that care about "have I reconciled *this*
spec yet?" stamp the generation they last handled into status — usually as
`status.observedGeneration` on the object, and often again as
`observedGeneration` on individual conditions.

So the check is boring and important:

```yaml
status:
  observedGeneration: 3
  conditions:
    - type: Ready
      status: "True"
      observedGeneration: 3
      reason: Reconciled
      message: desired state matches
```

If `metadata.generation` is `4` and that condition still says
`observedGeneration: 3`, the Ready status is about the *previous* spec.
Treat it as stale until a controller rewrites it for generation 4.

{{< note "gotcha" >}}
Not every resource or condition carries `observedGeneration` — and some
controllers only set it on the object, not on each condition. Absence isn't
the same as a match; it's just less information. When the field *is*
present and it doesn't equal `metadata.generation`, don't trust the
condition yet.
{{< /note >}}

Same idea shows up in custom resources and built-ins that follow the
API conventions: generation is the version of the desired state;
`observedGeneration` is how far status has caught up. If they diverge,
status is lagging — not lying, just not about *this* generation.
