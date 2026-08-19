---
title: "TIL: matchPolicy: Equivalent means your webhook still runs on versions you never listed"
date: 2026-08-19
---
I assumed a webhook registered against one API version simply didn't see
requests on any other version. Turns out that's only true if you set
`matchPolicy: Exact` — and that's not the default.

A `MutatingWebhookConfiguration` declares `rules` for the group/version/
resource it cares about:

```yaml
rules:
  - apiGroups: ["example.io"]
    apiVersions: ["v1beta1"]
    resources: ["widgets"]
```

Fine if the resource only has one version. But plenty of resources — CRDs
especially — have several, wired together by a conversion webhook. What
happens on a version your `rules` didn't list is exactly what
`matchPolicy` decides.

**`Exact`** — only the versions literally named in `rules` get sent to the
webhook. Anything else is skipped, as if the webhook doesn't exist.

**`Equivalent`** (the default) — the API server treats versions of the
same resource as interchangeable, provided a conversion webhook can get
between them. A request on a version you never listed still reaches your
webhook: the object gets converted into a version you *did* list, your
webhook runs, and the result gets converted back before it's returned to
the client.

So a webhook written against `v1beta1` alone quietly starts catching
`v1beta2` requests too, via a convert → webhook → convert-back round trip
that never shows up anywhere in your `rules`.

{{< note "gotcha" >}}
The object your webhook operates on may not be the object the client
actually sent — it's a converted stand-in. If your conversion functions
aren't perfectly lossless in both directions, `Equivalent` routes traffic
through that lossy path with zero indication a conversion ever happened.
{{< /note >}}

`Equivalent` is convenient — write the webhook once, keep it working as
the resource grows more versions. That's why it's the default. But if a
webhook only lists one version, don't assume the other versions are being
ignored. Check `matchPolicy` first.
