---
title: "TIL: Argo CD's dry-run can fail before your manifests even get a chance"
date: 2026-08-11
---
I've hit this exact failure on so many Argo apps that I probably should
have figured out what was actually going on years ago: a CRD and a custom
resource that depends on it, installed in the same sync, blowing up
before either one gets a real chance.

Before Argo CD actually applies anything, it runs each resource through
`kubectl apply --dry-run=server` as a sanity check. That's normally a good
thing — it catches broken manifests before they touch the cluster. But the
dry-run for the CR can hit the API server before the CRD has registered.
The API server has no idea what kind you're talking about, so it rejects
the dry-run with something like `no matches for kind "Foo" in version
"v1"`. The error reads like a Kubernetes problem — because it is one — but
it's easy to go looking for it in the chart templating first.

The fix is a sync option that tells Argo CD not to bother dry-running a
resource whose kind isn't registered on the cluster yet, and just attempt
the real apply instead:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  syncPolicy:
    syncOptions:
      - SkipDryRunOnMissingResource=true
```

{{< note "sync waves still matter" >}}
This only papers over the dry-run step. The real apply will still fail if
the CRD isn't actually installed by the time the CR gets applied — you
still need the CRD in an earlier sync wave (or as a sync hook) so it lands
first.
{{< /note >}}

Once the CRD is registered, the dry-run has something to check against
again and behaves normally. Everything else about validation stays the
same — this just stops Argo from failing a resource for the crime of
depending on something that's installing in the same breath.
