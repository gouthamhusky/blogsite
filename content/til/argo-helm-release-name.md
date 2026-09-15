---
title: "TIL: argo cd helm release name defaults to the Application name"
date: 2026-09-15
---
I kept looking for where the Helm release name came from in an Argo CD
Application — chart values, `helm.sh/release-name`, something in the
destination. It's simpler than that.

Unless you set `spec.source.helm.releaseName`, Argo CD uses the
Application's `metadata.name` as the Helm release name. Rename the App,
you rename the release. Want a stable release name that doesn't track the
App object? Override it explicitly.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: team-a-guestbook
  namespace: argocd
spec:
  project: default
  source:
    chart: guestbook
    repoURL: https://example.com/charts
    targetRevision: 1.2.3
    helm:
      releaseName: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
```

Without that `releaseName`, this would ship as release `team-a-guestbook`
because that's the Application name — fine until you care about matching
an existing Helm release or keeping `.Release.Name` short and stable.

{{< note "gotcha" >}}
Overriding the release name can bite charts that key off
`app.kubernetes.io/instance`. Argo CD still injects that label with the
*Application* name for tracking, so after an override the release name and
the instance label diverge — selectors that expected them to match can
break. Fix path is usually `application.instanceLabelKey` in `argocd-cm`,
not guessing at the chart.
{{< /note >}}

So when `helm list` shows a release that looks weirdly like your App name:
that's the default. Set `spec.source.helm.releaseName` when you want
something else.
