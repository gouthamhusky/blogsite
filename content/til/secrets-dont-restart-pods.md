---
title: "TIL: Kubernetes doesn't restart pods when a Secret changes — sometimes it doesn't even need to"
date: 2026-08-14
---
I assumed updating a Secret would eventually reach every pod using it, one
way or another. It doesn't — and the two ways pods consume Secrets behave
completely differently once the Secret changes.

If you're pulling a Secret in as an **environment variable**, it's
resolved once at pod creation and frozen. Update the Secret, and the
running container's environment never changes. Nothing short of a new pod
picks up the new value.

If you're **mounting** the Secret as a volume, though, the file on disk
*does* update — kubelet syncs it, usually within about a minute.
Kubernetes just doesn't restart anything for you; only an app that's
actually watching the file for changes will notice.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
type: Opaque
stringData:
  password: hunter2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      containers:
        - name: api
          image: api:latest
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-creds
                  key: password
          volumeMounts:
            - name: creds
              mountPath: /etc/creds
      volumes:
        - name: creds
          secret:
            secretName: db-creds
```

Same Secret, two consumption paths in the same container — `DB_PASSWORD`
is frozen forever, `/etc/creds/password` quietly updates on its own.

{{< note "gotcha" >}}
Volume auto-update doesn't happen if the mount uses `subPath` — those
files never refresh after pod start, full stop. And a Secret marked
`immutable: true` opts out of the update watch entirely (a real perf win
at scale), which also means you can't update it in place at all — delete
and recreate instead.
{{< /note >}}

Either way, if you actually want the app to pick up new values, you're
rolling the deployment yourself: `kubectl rollout restart deployment/api`,
or letting something like Reloader watch for you.
