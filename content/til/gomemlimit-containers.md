---
title: "TIL: GOMEMLIMIT makes Go runtime container-aware"
date: 2026-08-28
---
Go's garbage collector has no idea it's running inside a container. Without
telling it otherwise, it makes collection decisions purely based on heap
growth — specifically, it targets collecting when the heap doubles relative
to the size at the last collection. That works fine on a bare machine with
plenty of headroom, but in a container with a hard memory limit it can be
fatal: the heap grows to near the limit, the GC hasn't triggered yet because
the heap hasn't doubled, and the OOM killer arrives first.

`GOMEMLIMIT`, added in Go 1.19, solves this by giving the runtime an
explicit ceiling. As memory approaches the limit, the GC runs more
aggressively to stay below it.

```
GOMEMLIMIT=400MiB
```

It's a soft limit — the runtime won't panic if you momentarily exceed it,
it just tries harder to stay under. The convention is to set it to around
90% of your container's memory limit so there's headroom for non-heap
allocations (stack, runtime internals, cgo).

There are now two knobs:

- **`GOGC`** — controls the heap-doubling heuristic (default: `100`, meaning
  collect when heap reaches 2× the previous live set)
- **`GOMEMLIMIT`** — caps total memory usage and drives GC harder as you
  approach it

A pattern that's valid in production: turn `GOGC` off entirely and rely
solely on `GOMEMLIMIT`.

```
GOGC=off
GOMEMLIMIT=400MiB
```

With `GOGC=off`, the heap-doubling heuristic is disabled. GC only fires
when the process is approaching the memory limit. The tradeoff: you get
more predictable memory behavior and fewer unnecessary collections during
idle periods, at the cost of potentially running GC more frequently if
your working set is large and close to the limit.

{{< note "tip" >}}
If you use `GOGC=off` without `GOMEMLIMIT`, GC never runs until the
process is killed. Always set both together.
{{< /note >}}

For a Kubernetes deployment it looks like:

```yaml
env:
  - name: GOMEMLIMIT
    valueFrom:
      resourceFieldRef:
        resource: limits.memory
  - name: GOGC
    value: "off"
```

`resourceFieldRef` wires the limit directly from the pod spec so you don't
have to keep two numbers in sync.
