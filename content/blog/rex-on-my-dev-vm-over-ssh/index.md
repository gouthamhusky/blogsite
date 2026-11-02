---
title: "reaching rex on my dev vm over an ssh tunnel"
date: 2026-11-01
categories: ["linux"]
tags: ["ssh", "systemd", "launchd"]
draft: false
---
[Rex](https://www.superlogical.com/rex/docs/linux/installation) is a terminal multiplexer with a native Mac app. It's built as a client and a server: the server owns the sessions, and the app and the `rex` CLI are clients that connect to it. I wanted my sessions to live on my Linux dev VM, which means getting the app on my laptop to talk to a Rex server on the VM.

That turned into a nice small tour of `ssh -L`, launchd and systemd user services, so here it is written down.

## two things were wrong

I'd added the VM to Rex under the name I use with ssh, `devvm`, and it wouldn't connect. `rex hosts check` says why:

```
LABEL  STATUS                   ENDPOINT  DETAIL
devvm  not reachable from here  devvm     name does not resolve
```

Rex doesn't read `~/.ssh/config`. `devvm` is a `Host` alias that only ssh knows about, so Rex treated it as a hostname and asked DNS. For a bare name it tries HTTPS on 443, then HTTPS and HTTP on port 7297, and every one of those failed at the lookup.

Fixing the name wouldn't have helped anyway, because nothing was listening. A Rex server on Linux listens only on a Unix socket by default:

```
/run/user/1000/rex/server.sock
```

That's on purpose. Rex doesn't have real authentication yet, and [its docs](https://www.superlogical.com/rex/docs/linux/remote-access) are upfront about it: "Rex trusts a caller based only on how they connect today." The supported ways onto the network are Tailscale, which tells Rex who each caller is, or a plain HTTP listener with `--allow-anonymous`, which gives anyone who can reach the port full access. My VM doesn't run Tailscale, and I wasn't going to do the second one.

## ssh can forward unix sockets

I've used `ssh -L` to forward TCP ports for years. Until this I didn't know it forwards Unix sockets too, with the same `local:remote` syntax but paths instead of ports:

```bash
ssh -N -L /tmp/rex-devvm.sock:/run/user/1000/rex/server.sock devvm
```

ssh creates `/tmp/rex-devvm.sock` on the laptop. Every connection to it gets carried over the ssh connection, and on the VM, sshd connects to Rex's socket *as me*, because that's who I logged in as.

[![On the laptop, the Rex app and CLI connect to /tmp/rex-devvm.sock, which an ssh process kept running by launchd forwards over port 22 to sshd on the dev VM. sshd connects as you to /run/user/1000/rex/server.sock, and the Rex server, a systemd user service, lets the same user in. A red dashed line shows the direct rex -S devvm attempt failing with no such host.](diagram.svg)](diagram.svg)

That last part is why this needs no auth setup. The server sees a local caller running as the same user that owns the server, which is exactly who a default Rex server lets in. The socket is mode `0600`, so nobody else on the VM can open it, and the only way into the tunnel is my ssh key.

Rex even expects you to do this. `rex hosts add --help` says a socket endpoint "works only on a device where the path exists, such as the local end of an ssh -L forward." So:

```bash
rex hosts add devvm-tunnel unix:///tmp/rex-devvm.sock
rex -S devvm-tunnel ls
```

In the Mac app, *Add Remote Host…* takes the same `unix://` address, and the VM's sessions showed up.

## keeping the tunnel up with launchd

A tunnel I start by hand dies the first time the laptop sleeps or the VPN drops. On macOS, the thing that restarts processes for you is launchd, so the tunnel became a LaunchAgent:

{{< file "~/Library/LaunchAgents/local.rex-devvm-tunnel.plist" >}}
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>local.rex-devvm-tunnel</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/ssh</string>
    <string>-N</string>
    <string>-o</string><string>BatchMode=yes</string>
    <string>-o</string><string>ExitOnForwardFailure=yes</string>
    <string>-o</string><string>StreamLocalBindUnlink=yes</string>
    <string>-o</string><string>ConnectTimeout=15</string>
    <string>-o</string><string>ServerAliveInterval=30</string>
    <string>-o</string><string>ServerAliveCountMax=3</string>
    <string>-L</string>
    <string>/tmp/rex-devvm.sock:/run/user/1000/rex/server.sock</string>
    <string>devvm</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>ThrottleInterval</key>
  <integer>15</integer>
  <key>StandardErrorPath</key>
  <string>/Users/you/Library/Logs/rex-devvm-tunnel.log</string>
</dict>
</plist>
```
{{< /file >}}

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.rex-devvm-tunnel.plist
```

The ssh options do more work than they look like they do:

- **No `-f`.** My first tunnel was `ssh -fN`, which forks into the background. Under launchd that looks like the job exited. Stay in the foreground and let launchd supervise it.
- **`ExitOnForwardFailure=yes`.** Without it, ssh can stay connected with a forward that never got set up. With it, ssh exits and launchd tries again.
- **`StreamLocalBindUnlink=yes`.** After a restart the old `/tmp/rex-devvm.sock` is still on disk, and binding to it fails. This deletes it first.
- **`ServerAliveInterval=30` and `ServerAliveCountMax=3`.** After a sleep, the connection can be dead without either side noticing. Three missed keepalives, about 90 seconds, and ssh gives up so launchd can replace it.
- **`BatchMode=yes`.** Never prompt for anything. My key lives in 1Password's ssh agent, so with the vault locked ssh just fails, and launchd retries every 15 seconds (`ThrottleInterval`) until I unlock it.

To check that it actually comes back, I killed the ssh process. launchd had a new one up within about 20 seconds, and the Mac app reconnected without me touching it.

## keeping the server up with systemd

That left the server, which I'd started by hand with `rex server start`. Nothing was watching it.

The `rex` package ships a systemd unit, `rex.service`, and it was already running on the VM. The comments in the unit file say to grant access by adding people to the `rex` group, so I did that and pointed a tunnel at `/run/rex/server.sock`. It connected, and then every new terminal died immediately.

{{< note "gotcha" >}}
The packaged `rex.service` is a *shared* server. It runs as a locked `rex` system account, and every terminal it starts runs as that account, not as you. That account's shell is `nologin`, so a new terminal prints `This account is currently not available.` and exits. It's meant for a shared machine where the `rex` account is the security boundary, not for working as yourself.
{{< /note >}}

What I wanted was the server I'd been running by hand, but supervised. That's a systemd **user** unit. I copied the command line from the hand-started server (`ps` shows it), trimmed here:

{{< file "~/.config/systemd/user/rex.service" >}}
```ini
[Unit]
Description=Rex server (personal, runs as this user)

[Service]
Type=notify
ExecStart=/usr/bin/rex server run --listen unix://%t/rex/server.sock --log-format text
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=default.target
```
{{< /file >}}

`%t` is the user's runtime directory, `/run/user/$UID`, so the socket lands exactly where the hand-started server put it, and nothing on the laptop had to change. `Type=notify` works because Rex tells systemd when it's ready for connections. The packaged unit uses it too.

```bash
systemctl --user daemon-reload
rex server stop
systemctl --user enable --now rex.service
```

There's one more piece. Your user's systemd instance normally starts when you log in and stops when your last session ends, taking its services with it. **Linger** starts it at boot and keeps it running:

```bash
loginctl enable-linger
loginctl show-user $USER -p Linger   # Linger=yes
```

Mine was already on. Same test as before: I ran `kill -9` on the server, and systemd had a new one running in under 10 seconds. The tunnel and the app just reconnected.

The sessions were still listed after the restart too, which surprised me. I haven't dug into how much of a session survives, so I wouldn't count on a running program making it through a crash.

## where it ended up

- the Rex app and CLI talk to a socket on my laptop
- launchd keeps an ssh process forwarding that socket to the VM
- systemd keeps a Rex server running as me on the VM

No new ports, no new auth system, nothing new listening on the network. The trust boundary is my ssh key, same as it was before.

What I haven't tested yet is a VM reboot. With linger on and `WantedBy=default.target`, the server should come back by itself, and the laptop side will reconnect on its next retry.
