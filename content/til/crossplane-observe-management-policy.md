---
title: "TIL: crossplane Observe means it does not update the cloud resource"
date: 2026-09-11
---
I kept reading `managementPolicies: ["Observe"]` as "Crossplane is watching
this, so of course it'll keep the cloud resource in sync." Nope. Observe
is read-only.

When a managed resource's `managementPolicies` is just `Observe`, Crossplane
imports / syncs the *status* of the external resource into the Kubernetes
object — `status.atProvider` gets filled in — but it will not create,
update, or delete anything in the cloud. Spec changes in the MR sit there
looking intentional; the provider will not push them.

```yaml
apiVersion: example.crossplane.io/v1beta1
kind: Bucket
metadata:
  name: already-exists
spec:
  managementPolicies: ["Observe"]
  forProvider:
    region: us-west-2
```

That's the whole point for importing existing infrastructure: you can pull
a live resource into the cluster, look at it, reference it, without risking
a surprise Update on the first reconcile.

The default is `["*"]` — observe, create, update, late-initialize, and
delete. Individual policies (`Create`, `Update`, `Delete`, `LateInitialize`,
`Observe`) can be mixed. Dropping `Update` (or only listing `Observe`) is
exactly how you tell Crossplane "look, don't touch."

{{< note "gotcha" >}}
`Observe` alone also means Crossplane won't delete the cloud resource if
you delete the managed resource. If you later flip to `["*"]` without
copying the right values from `status.atProvider` into `spec.forProvider`,
the next reconcile can try to "fix" drift you never meant to enforce.
{{< /note >}}

So when someone says "it's on Observe," translate that as: Crossplane can
see the cloud resource. It will not update it.
