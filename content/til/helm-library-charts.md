---
title: "TIL: Library Helm charts share templates, not workloads"
date: 2026-10-30
---

Every chart I work on ends up with the same handful of helper templates: standard labels, a fullname helper, the same resource-limit boilerplate. I used to copy them from chart to chart. Library charts are Helm's answer to that.

A library chart is a chart with `type: library` in its `Chart.yaml`:

```yaml
apiVersion: v2
name: common
version: 0.1.0
type: library
```

Instead of manifests, it only ships named templates — the `define` blocks you'd normally keep in `_helpers.tpl`:

```yaml
{{- define "common.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

An application chart pulls it in as a regular dependency:

```yaml
# the app chart's Chart.yaml
dependencies:
  - name: common
    version: 0.1.0
    repository: file://../common
```

and then calls the helpers like its own:

```yaml
metadata:
  labels:
    {{- include "common.labels" . | nindent 4 }}
```

The library chart itself deploys nothing. It has no rendered manifests of its own, and Helm refuses to install it standalone — it exists purely to be depended on.

{{< note "gotcha" >}}
Template names are global once loaded. If two library charts both define `labels`, one silently wins, so prefix every `define` with the library chart's name (`common.labels`, not just `labels`). Also, helpers execute in the caller's context: `.Values` inside a library template reads the *parent* chart's values, not defaults from the library chart's own `values.yaml`.
{{< /note >}}
