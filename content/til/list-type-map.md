---
title: "TIL: making CRD lists behave like maps under server-side apply"
date: 2026-08-06
---
I'd been quietly assuming Kubernetes lists were just lists. Turns out some
of them are undercover maps, and not knowing which is which explains a
whole class of server-side apply conflicts I used to debug by vibes.

Here's the actual mechanism: when a CRD field is a list of objects, SSA
treats the whole list as one atomic value by default. It doesn't know
entry 3 in my update and entry 3 in yours are supposed to be different
things — as far as it's concerned we're both just yelling into slot 3. Two
controllers each adding or updating their own entry look like they're
fighting over the same field, so you get spurious conflicts, or one write
silently clobbering the other.

You can tell the API server to treat the list like a map instead, keyed by
whichever field(s) uniquely identify an entry:

```yaml
type: array
x-kubernetes-list-type: map
x-kubernetes-list-map-keys:
  - type
items:
  type: object
  properties:
    type:
      type: string
    status:
      type: string
```

{{< note "controller-gen" >}}
If your CRD comes from Go types, you don't hand-write that YAML at all —
just slap these on the field and let `controller-gen` generate it for you:

```go
// +listType=map
// +listMapKey=type
Conditions []Condition `json:"conditions"`
```
{{< /note >}}

With both markers set, SSA merges by key instead of by index — each
controller can own and update its own entry (`type: Ready`,
`type: Progressing`, ...) without touching anyone else's. The key fields
have to be `required` and effectively immutable, or the schema gets
rejected.

This is exactly how `status.conditions` works on most built-in types —
turns out I'd been benefiting from this for years without knowing what
made it possible. 🤦
