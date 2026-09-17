---
title: "TIL: a DNS zone delegation is just NS records (plus glue when you need them)"
date: 2026-09-17
---
I used to picture "zone delegation" as some special registry ceremony.
It's mostly NS records at a zone cut — the parent pointing resolvers at
the child's authoritative name servers.

When `example.com` is delegated from `.com`, the parent publishes an NS
set for `example.com`. A recursive resolver that hits that cut stops
asking the parent for answers and asks those child name servers instead.
The child zone also has its own apex NS (and SOA); that's the
authoritative copy. The parent's NS set is a referral, not the child's
truth.

```text
; at the parent (e.g. .com)
example.com.    NS  ns1.example.com.
example.com.    NS  ns2.example.com.

; glue — needed because ns1/ns2 live *inside* the delegated zone
ns1.example.com.  A     203.0.113.10
ns2.example.com.  AAAA  2001:db8::10
```

That glue matters. If the name servers are in-bailiwick
(`ns1.example.com` serving `example.com`), the resolver can't learn their
addresses by following the delegation first — chicken and egg. So the
parent returns A/AAAA glue alongside the NS referral. Out-of-bailiwick
servers (`ns1.provider.net`) don't need that glue from this parent; those
names resolve through a different path.

So "we delegated the zone" usually means: NS at the parent, matching
authoritative NS at the child, and glue when the NS hostnames sit inside
the child. Miss any of those and resolution stalls in ways that look like
mystery timeouts rather than a missing pointer.
