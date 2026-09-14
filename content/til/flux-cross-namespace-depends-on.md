---
title: "TIL: flux dependsOn can order HelmReleases and Kustomizations across namespaces"
date: 2026-09-14
---
I assumed Flux dependency ordering was same-namespace only — put everything
in `flux-system` or give up on `dependsOn`. Nope. Both `Kustomization` and
`HelmRelease` let you point `spec.dependsOn` at a peer in another namespace.

Add `namespace` on the dependency reference. The controller waits until that
object is `Ready` before reconciling yours — even when it lives somewhere
else.

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: webapp
  namespace: team-a
spec:
  dependsOn:
    - name: infrastructure
      namespace: flux-system
  interval: 10m
  path: ./apps/webapp
  prune: true
  sourceRef:
    kind: GitRepository
    name: fleet-infra
    namespace: flux-system
```

Same shape on a `HelmRelease`: `dependsOn` with `name` + `namespace` gates
one release on another across namespaces. Handy when infra lives in a
shared NS and apps are tenant-scoped.

{{< note "gotcha" >}}
Cross-namespace is not cross-kind. A `Kustomization` can only depend on
other `Kustomization`s; a `HelmRelease` only on other `HelmRelease`s. To
order a Kustomization after a Helm chart, wrap the release in its own
Kustomization (`wait: true`) and depend on that. Also: multi-tenant
clusters may set `--no-cross-namespace-refs=true` and shut this off.
{{< /note >}}

So when apps and infra aren't co-located: you don't have to collapse
namespaces just to get ordering. Set `dependsOn[].namespace` and let Flux
wait across the boundary.
