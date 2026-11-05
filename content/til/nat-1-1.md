---
title: "TIL: 1:1 NAT maps addresses, not ports"
date: 2026-11-04
---

Most NAT I deal with day to day is the many-to-one kind: a whole subnet shares one public IP, and the router keeps everyone apart by rewriting port numbers. 1:1 NAT is the simpler, more wasteful cousin. One private address maps to exactly one public address, statically, for every port:

```
10.0.3.17   <->   203.0.113.17
```

Outbound, the source IP gets rewritten to the public address. Inbound, anything arriving at the public address gets its destination rewritten to the private one. Ports pass through untouched — there's no table of port mappings to maintain, just one address pair. With iptables it's literally one rule each way:

```bash
iptables -t nat -A PREROUTING  -d 203.0.113.17 -j DNAT --to-destination 10.0.3.17
iptables -t nat -A POSTROUTING -s 10.0.3.17    -j SNAT --to-source    203.0.113.17
```

An AWS Elastic IP is this same idea: the instance only ever sees its private address, and the internet gateway does the 1:1 translation at the edge.

The trade is obvious once you see it. You burn a whole public IPv4 address per host, but in return the host is reachable inbound on any port and always egresses from the same source IP — which is exactly what you want when the other side is allowlisting you, or a service needs a stable public identity without a load balancer in front of it.

{{< note "gotcha" >}}
The host never knows its public address — it only sees the private one. Anything that advertises its own address in the payload (active-mode FTP, some SIP/VoIP setups, cluster members exchanging peer addresses) will happily hand out the useless private IP unless you configure the public one explicitly. Also, NAT is not filtering: 1:1 mapping makes every port reachable in principle, so the firewall rules in front of it are doing all the actual protecting.
{{< /note >}}
