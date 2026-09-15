---
title: "TIL: deleting a CRD deletes every instance of that kind across the cluster"
date: 2026-09-15
---
I used to treat CRD deletion like uninstalling an operator binary — remove
the type definition, leave the objects alone, clean up later. Kubernetes
doesn't work that way.

When you delete a CustomResourceDefinition, the API server garbage-collects
*all* custom resources of that kind in *every* namespace. Not just the ones
in the same namespace as the CRD (CRDs are cluster-scoped anyway). The whole
inventory for that API group/kind goes with it.

```sh
kubectl delete crd certificates.cert-manager.io
# every Certificate object in every namespace is gone too
```

That's by design: without the CRD, the API server has no schema to store or
serve those objects under. So "delete the CRD to reinstall cleanly" is a
cluster-wide data wipe for that kind, not a local reset.

Same trap shows up when Helm or an operator chart owns the CRD and an
uninstall/upgrade removes it — the instances leave with the definition.
Before you `kubectl delete crd …`, assume every live object of that kind
is about to disappear.
