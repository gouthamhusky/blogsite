---
title: "TIL: crossplane's external-name annotation is the cloud resource id"
date: 2026-09-11
---
I used to assume the Kubernetes `metadata.name` on a Crossplane managed
resource was what the cloud saw too. It isn't — at least not always.

Providers key off the `crossplane.io/external-name` annotation. That value
is the name (or id) of the resource *inside the provider*. The provider
looks it up to decide whether the external thing already exists, and if you
set it before create, that value becomes the external name instead of
defaulting to the Kubernetes object name.

```yaml
apiVersion: rds.aws.m.upbound.io/v1beta1
kind: Instance
metadata:
  name: my-rds-instance
  annotations:
    crossplane.io/external-name: my-custom-name
spec:
  forProvider:
    region: us-west-2
```

Here the MR is `my-rds-instance` in the cluster, but AWS gets
`my-custom-name`. Same annotation is how you import something that already
exists: point `external-name` at the live id, usually with
`managementPolicies: ["Observe"]` first so you don't write before you're
ready.

When the provider generates the name itself, it tries to write that back
into the annotation. If it can't, you've leaked a cloud resource with no
pointer on the MR.

{{< note "gotcha" >}}
A typo in `crossplane.io/external-name` (or missing identifying
`forProvider` fields like region) won't adopt the resource you meant —
the provider may create a *new* one. And Crossplane doesn't enforce
cluster-wide uniqueness on the annotation, so two managed resources with
the same external-name can fight over one cloud object.
{{< /note >}}

So when you're staring at an MR and wondering which cloud thing it owns,
read `crossplane.io/external-name` first — not `metadata.name`.
