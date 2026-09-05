# Palo Alto Segmentation Lab

A zone-based segmentation design on Palo Alto PAN-OS: security zones, least-privilege
policy, App-ID instead of port rules, and East-West control between segments that most
networks leave flat. Built to make the Palo Alto SASE and Cloud Security certifications
show up as something a hiring manager can read, not just a line on a résumé.

> The policy in this repo is expressed as PAN-OS **set commands** and as an annotated rule
> table. It is designed against the PA-VM (VM-Series) trial and is deployable as-is in a lab.
> No real IPs, no real keys — see [Handling of secrets](#handling-of-secrets).

---

## The problem this design addresses

Most networks are hard on the outside and flat on the inside. Once something lands in the
user VLAN, it can reach the servers, the databases, the management interfaces — because the
internal firewall, if there is one, filters North-South and lets East-West through. Ransomware
and lateral movement live in exactly that gap.

This lab segments the inside. Every zone-to-zone flow is denied by default and opened only for
a named application between named hosts. A compromised workstation can reach the one service
it is supposed to use and nothing else.

## Zone model

```
                          [ PA-VM firewall ]
        TRUST         DMZ            SERVERS         MGMT           UNTRUST
      (users)     (public web)    (app+db tier)   (device mgmt)   (internet)
      10.10.0/24   10.20.0/24      10.30.0/24      10.99.0/24       WAN

  Default intrazone: allow      Default interzone: DENY (explicit, logged)
```

Five zones, and the interesting rules are the ones **between internal zones**, not the ones
facing the internet:

- `TRUST -> SERVERS` — users reach the app tier on one application, not the database directly
- `DMZ -> SERVERS` — the web front-end reaches the app tier and nothing back toward users
- `SERVERS -> SERVERS` — the app tier reaches the DB tier; the DB tier initiates nothing
- everything `-> MGMT` — denied except from the admin jump host
- East-West between user subnets — denied (micro-segmentation)

## What is in here

| Path | Contents |
|---|---|
| [`docs/01-design.md`](docs/01-design.md) | Zone rationale, the trust model, why default-deny interzone |
| [`docs/02-appid-vs-ports.md`](docs/02-appid-vs-ports.md) | Why App-ID rules beat port rules, with the failure mode of each |
| [`docs/03-east-west.md`](docs/03-east-west.md) | The micro-segmentation rules and the lateral-movement they stop |
| [`policy/security-policy.md`](policy/security-policy.md) | The full rule base as an annotated table |
| [`policy/pan-os-set-commands.txt`](policy/pan-os-set-commands.txt) | The same policy as deployable `set` commands |
| [`topology/lab-setup.md`](topology/lab-setup.md) | Building it on the PA-VM trial |

## The design principles it demonstrates

1. **Default-deny interzone, and the deny is explicit and logged.** The implicit deny at the
   bottom of a PAN-OS rule base does not log by default — so an explicit final deny-and-log
   rule is what turns "it's blocked" into "we can see what tried".
2. **App-ID, not ports.** A rule that permits `tcp/443` permits anything on 443. A rule that
   permits `ssl` and `web-browsing` permits the web and blocks the tunnel someone hid on 443.
3. **Least privilege per flow.** Every allow rule names a source zone, a destination zone, a
   source, a destination, and an application. No `any` in a production allow rule.
4. **Logging on every rule.** A rule that does not log is a blind spot the day you need it.

## Handling of secrets

No admin password, API key, certificate or real public IP is committed. Addresses are from
RFC 1918; the untrust interface uses a documentation address. Deploy-time secrets are
`<PLACEHOLDER>` values.

## Author

Kodjo Apedoh — Network & Cloud Security · Arlington, VA
CCNA · Fortinet NSE · **Palo Alto SASE & Cloud Security** · [LinkedIn](https://www.linkedin.com/in/kodjo-apedoh-03030990/) · [Other labs](https://github.com/ktf40858-stack)

## License

MIT — see [LICENSE](LICENSE). Lab and educational use only.
