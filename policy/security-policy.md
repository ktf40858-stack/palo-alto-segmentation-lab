# Security policy — annotated rule base

The rule base read top to bottom. PAN-OS is **first-match-wins**, so order is part of the
policy, not a detail. Specific allows sit above the explicit deny at the bottom.

| # | Name | From | To | Source | Dest | App | Action | Log | Why |
|---|---|---|---|---|---|---|---|---|---|
| 1 | users-to-app | TRUST | SERVERS | users | app host | web-browsing, ssl | allow | yes | Users reach the app front-end only — never the DB directly |
| 2 | web-to-app | DMZ | SERVERS | web host | app host | web-browsing, ssl | allow | yes | The public web tier talks to the app tier, one direction |
| 3 | app-to-db | SERVERS | SERVERS | app host | db host | mssql-db | allow | yes | App tier reaches the DB; DB initiates nothing |
| 4 | outbound-web | TRUST, DMZ | UNTRUST | users, dmz | any | web-browsing, ssl, dns | allow | yes | Internet access, App-ID filtered and threat-inspected |
| 5 | mgmt-access | TRUST | MGMT | jump host | servers | ssh, ms-rdp | allow | yes | Device management from the admin jump host only |
| 6 | **deny-all-log** | any | any | any | any | any | **deny** | **yes** | Explicit default-deny — stops East-West, and logs what tried |

## Reading the design out of the table

**There is no `any` in an allow rule's application column.** Every permit names the
application. Rule 1 allows `web-browsing` and `ssl` to the app host — not `tcp/443`, not
`any`. A reverse shell that a compromised workstation tries to open to the app host on 443
is not `ssl` to App-ID, so it does not match rule 1, falls through to rule 6, and is denied
and logged.

**`service application-default` is doing quiet work.** It pins each application to the port
IANA assigns it. Without it, `web-browsing` would be allowed on *any* port, which reopens the
hole App-ID was meant to close. App-ID plus application-default is the pair; either one alone
leaks.

**The database is a sink, never a source.** Rule 3 lets the app tier reach the DB. Nothing
lets the DB reach anything. A compromised database server cannot phone home or move laterally,
because every flow it initiates hits rule 6.

**Rule 6 is the whole point, and it is explicit on purpose.** PAN-OS has an implicit interzone
deny at the bottom, but it does not log. Writing the deny explicitly with `log-end yes` means
every blocked lateral-movement attempt lands in the traffic log with a source, a destination,
and the application that was tried — which is what a SOC needs to know it is under attack, not
just that the network is quiet.

## What this stops, concretely

| Scenario | Rule that stops it |
|---|---|
| Compromised user PC scans the server subnet | 6 — no rule permits users to arbitrary server hosts |
| Compromised user PC connects straight to the DB | 6 — rule 1 only reaches the app host, on web apps |
| Malware on the DB server beacons to the internet | 6 — nothing permits SERVERS -> UNTRUST |
| Attacker hides C2 on tcp/443 to the app host | 6 — traffic is not App-ID `ssl`, application-default rejects the port mismatch |
| Lateral RDP from user subnet to a server | 6 — only the jump host reaches MGMT, and only on ssh/rdp |

## Best-practice profile group

Rules 1 and 4 attach the `best-practice` security profile group (Antivirus, Anti-Spyware,
Vulnerability Protection, URL Filtering, WildFire, File Blocking). Segmentation controls
*where* traffic may go; the profiles inspect *what* is inside the traffic that is allowed. A
segmentation lab without threat profiles on the allow rules is only half the control — the
allowed flows are exactly where the payload rides in.
