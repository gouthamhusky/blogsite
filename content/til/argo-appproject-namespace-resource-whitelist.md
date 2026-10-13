---
title: "TIL: Argo CD AppProjects have a namespace resource whitelist"
date: 2026-10-13
---

An AppProject doesn't just control *where* an Application can deploy (destinations) — it also controls *what kinds* of namespaced resources the apps in that project may manage, via `spec.namespaceResourceWhitelist`:

```yaml
spec:
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: ConfigMap
```

If an Application tries to sync a namespaced resource whose group + kind isn't on that list, the sync is rejected — even if the destination namespace itself is allowed. Cluster-scoped resources go through the separate `clusterResourceWhitelist`, so a `ClusterRole` won't be saved by listing it here.

{{< note "gotcha" >}}
The whitelist is kind-level, not name-level. There's no way to say "only this one ConfigMap named `foo`" — once `ConfigMap` is whitelisted, every ConfigMap in the project is fair game.
{{< /note >}}
