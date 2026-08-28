---
title: "TIL: Go's GC has no idea it's running inside a container — unless you tell it"
date: 2026-08-28
---
I assumed the Go runtime would respect the container's memory limit the
same way it respects available RAM on a bare machine. It doesn't — and the
difference is exactly what gets you OOM-killed.

By default, Go's GC collects when the live heap doubles relative to what
was alive at the last collection. That heuristic works fine with room to
grow. In a container with a hard ceiling, though, the heap can climb to
within a few MB of the limit without ever triggering a collection — because
it hasn't doubled yet. The OOM killer gets there first.

`GOMEMLIMIT`, added in Go 1.19, gives the runtime an explicit ceiling. As
memory approaches the limit, the GC runs more aggressively to stay under
it:

```
GOMEMLIMIT=400MiB
```

It's a soft limit — a momentary spike past it won't panic, the runtime
just tries harder to pull back. The convention is to set it around 90% of
the container's memory limit to leave headroom for the stack, runtime
internals, and cgo allocations that live outside the heap.

There are now two knobs controlling GC behavior:

- **`GOGC`** — the heap-doubling heuristic (default `100`: collect when
  heap reaches 2× the previous live set)
- **`GOMEMLIMIT`** — caps total memory and drives GC harder as you approach
  it

A pattern worth knowing: turn `GOGC` off entirely and let `GOMEMLIMIT` do
all the work.

```
GOGC=off
GOMEMLIMIT=400MiB
```

With `GOGC=off`, GC only fires when the process is approaching the limit —
no spurious collections mid-request just because the heap happened to
double. You trade slightly more aggressive GC near the ceiling for far more
predictable behavior everywhere else.

{{< note "gotcha" >}}
`GOGC=off` without `GOMEMLIMIT` means GC never runs. The two have to go
together.
{{< /note >}}

In Kubernetes you can pull the limit directly into the container using the
**Downward API**, so the two numbers never drift apart:

```yaml
env:
  - name: GOMEMLIMIT
    valueFrom:
      resourceFieldRef:
        resource: limits.memory
  - name: GOGC
    value: "off"
```

`resourceFieldRef` is how the Downward API exposes pod resource fields as
environment variables — no hardcoding, no separate ConfigMap to keep in sync.
