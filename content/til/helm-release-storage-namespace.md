---
title: "TIL: a Helm release's namespace is part of its identity"
date: 2026-10-02
---

A Helm release isn't just a name — it's a name plus the namespace its state lives in. Helm persists every release revision as a Secret (`sh.helm.release.v1.<name>.v<revision>`) in the release's storage namespace, which is also how `helm history` and `helm rollback` work:

```bash
kubectl get secrets -n flux-system -l owner=helm
# sh.helm.release.v1.myapp.v12
# sh.helm.release.v1.myapp.v13
# ...
```

GitOps tools make the split explicit: Flux's `HelmRelease` deploys workloads to a `targetNamespace` while keeping those release Secrets in a separate `storageNamespace` (often `flux-system`).

So changing the storage namespace isn't a move — Helm looks for the release in the new namespace, finds nothing, and treats it as brand new. The old release gets uninstalled from the previous namespace and the chart installs from scratch in the new one: revision 1, no history carried over.

{{< note "gotcha" >}}
The fresh install starts at revision 1 with zero rollback history. Anything that depends on release history — `helm rollback`, revision-pinned automation, or diffing against the previous release — breaks across the move. And if the old release's uninstall is skipped or fails, its resources stay orphaned in the old namespace with nothing tracking them.
{{< /note >}}
