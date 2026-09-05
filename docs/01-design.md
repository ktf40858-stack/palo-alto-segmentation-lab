# Design and trust model

## The starting assumption

Assume the inside is already breached. Not as pessimism — as the design premise. A workstation
will click the link, a vendor laptop will bring something in, a phone on the guest network will
be compromised. The question a segmentation design answers is not "how do we keep them out" but
"when they are in one zone, what can they reach from there". The answer should be: almost nothing.

## Why zones, not subnets

A subnet is a Layer 3 boundary; a zone is a policy boundary. PAN-OS attaches policy to zones,
so the zone is where the trust decision lives. Grouping interfaces into zones by **trust level
and function** — users, public-facing DMZ, internal servers, management — means the policy reads
as a trust statement: "the users zone may reach the app tier, on the app, and nothing else."

Five zones here, chosen so that each holds things with the same trust level and the same job:

| Zone | Holds | Trust |
|---|---|---|
| TRUST | user workstations | low — this is where compromise starts |
| DMZ | internet-facing web front-end | low — exposed by definition |
| SERVERS | app tier and DB tier | high — the assets |
| MGMT | device management interfaces | highest — keys to the kingdom |
| UNTRUST | the internet | none |

MGMT is separate from SERVERS deliberately. The management plane is a higher-value target than
the data it manages — an attacker who reaches a server steals data; one who reaches the
management interface owns the infrastructure. It gets its own zone and the tightest rule.

## Default-deny interzone

The single most important line in the whole design is that traffic **between** zones is denied
unless a rule permits it. PAN-OS gives you an implicit allow *within* a zone (intrazone) and an
implicit deny *between* zones (interzone). The design leans on both:

- Intrazone allow means hosts inside a zone talk freely — fine, they share a trust level.
- Interzone deny means every boundary crossing is a deliberate, named, logged exception.

The number of allow rules is therefore the size of the attack surface. Five allow rules means
five permitted boundary crossings. Anything not on that list does not happen.

## Making the deny explicit

The implicit interzone deny does not log. That is a real gap: the day you are breached, the
traffic log is where you reconstruct what the attacker touched, and an unlogged deny leaves that
reconstruction blind. So the design ends with an **explicit** deny-all rule with logging on. It
changes nothing about what is blocked; it changes everything about what you can see.

## Least privilege, applied per flow

Each allow rule is as narrow as the flow it serves:

- **Named source and destination**, not `any`. Rule 1 is `users -> app host`, not `users ->
  servers`. The DB host is in the same zone as the app host and is deliberately unreachable.
- **Named application**, not a port. `web-browsing` and `ssl`, not `tcp/443`.
- **application-default service**, so the app is pinned to its real port.

The result is that a rule permits exactly one intended conversation and no more. This is more
work to write and far more valuable to operate: when something breaks, the rule tells you
precisely what was and was not allowed, and when something attacks, it hits the deny.

## Where this design goes next

This is macro-segmentation with a first cut of micro-segmentation (the app/DB split inside the
SERVERS zone). The next step is full micro-segmentation — each workload its own segment, policy
following the workload rather than the subnet — which is where this design meets Zero Trust and
SASE. That is the subject of the [zero-trust-sase-architecture](https://github.com/ktf40858-stack/zero-trust-sase-architecture) repo.
