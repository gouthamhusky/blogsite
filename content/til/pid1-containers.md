---
title: "TIL: why containers need a real init process"
date: 2026-08-18
---
I'd always waved away the "use tini" advice in Dockerfiles as boilerplate
cargo-culting. Turns out PID 1 is genuinely special to the kernel, and
skipping it is exactly why containers hang on shutdown and quietly
accumulate zombie processes.

Two things are true about PID 1 that aren't true for any other process:

1. It's immune to the default action of unhandled signals. For every
   other process, an unhandled `SIGTERM` just kills it. PID 1 ignores it
   unless the program explicitly installs a handler.
2. It's responsible for reaping orphaned children via `wait()` — if it
   doesn't, they pile up as zombies.

Most application binaries do neither, because they were never written to
be an init system. So when your image runs the app directly as PID 1:

```dockerfile
# before: the app itself is PID 1
FROM node:20
COPY . .
CMD ["node", "server.js"]
```

`docker stop` — or a pod's `SIGTERM` during termination — doesn't
actually stop it. Node never installed a `SIGTERM` handler, and PID 1
ignores the kernel's default action, so nothing happens until the grace
period expires and Kubernetes sends `SIGKILL`. Any subprocess the app
spawns and forgets about becomes a zombie nobody reaps.

The fix is a real PID 1 that knows how to forward signals and reap
orphans, with the app running as PID 2 instead:

```dockerfile
FROM node:20
COPY . .
ENTRYPOINT ["tini", "--"]
CMD ["node", "server.js"]
```

{{< note "gotcha" >}}
Docker has this built in — `docker run --init` (or `init: true` in a
Compose file) does the same thing without adding `tini` to the image.
Kubernetes has no equivalent field; you either bake `tini` into the image
yourself or pick a base image that already includes it.
{{< /note >}}

None of this shows up in local testing — `docker run -it` and friends
behave fine either way. It only bites under real signal handling and
process churn, which is exactly when you don't want to be debugging it.
