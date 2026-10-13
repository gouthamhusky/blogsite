---
title: "TIL: GOTOOLCHAIN=local stops Go from auto-upgrading toolchains"
date: 2026-10-12
---

I ran into a repo whose `go.mod` said `go 1.24` while my machine had 1.23, and instead of erroring out, the `go` command quietly downloaded a whole new toolchain and switched to it. That's the default `GOTOOLCHAIN=auto` behavior since Go 1.21: the `go` line in go.mod is treated as a minimum toolchain requirement, and the go command will fetch and run whatever version satisfies it.

Setting `GOTOOLCHAIN=local` turns that off. The go command always uses the toolchain it shipped with, and if go.mod asks for something newer, it fails loudly instead of downloading:

```bash
$ go env -w GOTOOLCHAIN=local
$ go build ./...
go: go.mod requires go >= 1.24 (running go 1.23.4)
```

{{< note "gotcha" >}}
`go env -w` writes to your user config, so this sticks for every repo on the machine. That's what you want for reproducible CI images, but on a dev laptop it means an innocent `git pull` on a repo that bumped its go version suddenly stops building until you upgrade Go yourself.
{{< /note >}}

I now set `GOTOOLCHAIN=local` anywhere I want the build to fail fast on toolchain drift rather than silently pulling a new compiler from the network.
