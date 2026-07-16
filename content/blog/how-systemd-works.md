---
title: "how does systemd actually start things?"
date: 2025-06-22
categories: ["linux"]
tags: ["systemd"]
---
I've typed `systemctl start` thousands of times without really knowing what happens after I hit enter. Here's what I pieced together.

## units all the way down

Everything systemd manages is a **unit**. A service is a unit, a mount is a unit, a timer is a unit. They're defined in plain text files, which is nice — you can read them.

```bash
systemctl cat sshd.service
```

## dependencies are the interesting part

The thing I didn't appreciate: systemd builds a dependency graph and starts things in parallel wherever it can. `Wants=` and `Requires=` describe *ordering intent*, and `After=` / `Before=` describe *actual ordering*.

{{< note >}}
`Requires=` without `After=` doesn't guarantee start order — it only guarantees the dependency is pulled in. This trips people up constantly.
{{< /note >}}

That's the bit I always forget, so now it's written down.
