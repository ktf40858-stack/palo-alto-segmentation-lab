# East-West control and lateral movement

## The direction that gets forgotten

Firewalls were built to police North-South — the boundary between inside and outside. East-West
is the traffic *between* internal systems, and historically the firewall never saw it, because
internal systems shared a subnet and talked directly through the switch. That is the road every
intrusion travels: a single compromised host, then lateral movement across a flat interior to
reach the thing actually worth stealing.

Segmenting East-West means the internal firewall sits in the path between internal zones, and
the same default-deny that protects the perimeter now protects each internal boundary.

## The rules that do it here

Three flows are permitted inside the interior; everything else between internal hosts is denied.

```
TRUST  --web/ssl-->  app host        (rule 1)   users reach the app, nothing more
DMZ    --web/ssl-->  app host        (rule 2)   web tier reaches the app, one way
SERVERS --mssql-->   db host         (rule 3)   app reaches DB; DB reaches nothing
```

Read what is **absent**:

- No rule lets a user host reach another user host across subnets — a worm cannot spread
  laterally through the user population.
- No rule lets a user host reach the DB — the crown jewels are two zones and one unreachable
  hop away from where compromise starts.
- No rule lets the DB initiate anything — a compromised DB is a dead end, not a launch point.
- No rule lets the DMZ reach back toward users — a breached public web server cannot pivot
  inward to the workstations.

## A worked lateral-movement scenario

An attacker phishes a user and lands on a workstation in TRUST. From there:

1. **Scan the server subnet.** Every probe to a server host that is not the app host on a web
   app hits rule 6 (deny-all-log). The scan produces a wall of denies in the traffic log —
   which is itself the alert.
2. **Reach the app host** (the one thing permitted). Fine — but only on `web-browsing`/`ssl`,
   application-default. An exploit attempt against the app's management port is a different app
   on a different port: denied.
3. **Try to reach the DB directly.** No rule permits TRUST -> db host. Denied and logged.
4. **Try to open a C2 tunnel back out on 443.** Permitted zone (TRUST -> UNTRUST exists), but
   the traffic is not App-ID `ssl`/`web-browsing`, or it is, and the WildFire/Anti-Spyware
   profile on rule 4 catches the known C2 signature.

At every step the attacker is either denied or inspected. The flat-network version of this
scenario has the attacker at the database in step 2.

## Micro-segmentation: how far to take it

This lab micro-segments one boundary that matters — app tier from DB tier, inside the same
zone, via rule 3 with named hosts. Full micro-segmentation puts every workload in its own
segment with policy that follows the workload. On PAN-OS that means Dynamic Address Groups
keyed to tags (from VM attributes, from an orchestration system), so a workload's policy is
determined by *what it is*, not *where its IP happens to be*:

```
# a workload tagged 'role:database' is reachable only by 'role:app', by tag, wherever it lives
set address-group dag-db dynamic filter "'role.database'"
set address-group dag-app dynamic filter "'role.app'"
```

That is the bridge from classic segmentation to Zero Trust, and it is where this repo hands off
to [zero-trust-sase-architecture](https://github.com/ktf40858-stack/zero-trust-sase-architecture).

## The operational reality

East-West segmentation breaks things on day one, always — some app talks to some other app on a
port nobody documented, and the deny-all rule finds it. That is not a failure of the design; it
is the design surfacing an undocumented dependency. The rollout method is: deploy the allow
rules you know, put the deny rule in with logging but watch the log before you *rely* on it, and
promote flows from "seen in the log" to "explicit allow rule" until the deny is quiet. Skipping
that discovery phase and dropping straight to enforce is how a segmentation project gets rolled
back after it takes down a payroll run.
