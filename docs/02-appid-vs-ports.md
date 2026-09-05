# App-ID versus port-based rules

The difference between a legacy firewall and a next-generation one is one question: does the
rule match a **port**, or an **application**? It sounds like a detail. It is the whole ballgame.

## What a port rule actually says

`permit tcp/443` says "allow anything that uses port 443." It does not say "allow HTTPS."
Port 443 is where HTTPS lives, but it is also where every tool that wants to get through a
firewall has learned to live, precisely because everyone allows 443:

- a reverse shell tunnelled over TLS
- SSH tunnelled over 443 to bypass egress filtering
- a C2 channel dressed as HTTPS
- file exfiltration to a cloud drive on 443
- a VPN a user set up to get around policy

A port rule permits all of it, because all of it is "port 443."

## What an App-ID rule says

`allow ssl, web-browsing` says "allow traffic that is *actually* TLS or web browsing, on the
port those applications belong on." PAN-OS inspects the session, identifies the application from
its behaviour regardless of port, and matches the rule against the real application. Now:

- the reverse shell on 443 is identified as `ssh` or as an unknown tunnel, not `ssl` — denied
- SSH-over-443 is identified as `ssh` — denied by a web-only rule
- the C2 dressed as HTTPS either resolves to a known bad app or to `unknown-tcp` — denied

Same allowed business traffic, radically smaller hole.

## The trap: App-ID without application-default

App-ID alone is not enough. If a rule permits `web-browsing` with `service: any`, it permits
web-browsing **on any port** — and an attacker who runs their tunnel and gets it classified as
web-browsing on port 8443 walks straight through. The fix is `service application-default`,
which pins each application to the port(s) IANA assigns it. `web-browsing` then matches only on
80, `ssl` only on 443. App-ID says *what*, application-default says *where*, and you need both.

Every allow rule in this lab uses `application-default` for exactly this reason.

## The trap in the other direction: breaking your own apps

App-ID has a real cost: an internal application that runs on a non-standard port, or a custom
protocol App-ID does not recognise, will be classified as `unknown-tcp` and denied by an
App-ID rule base. The disciplined answer is a **custom App-ID** or an **application override**
for that specific known app — not widening a rule back to `any`. The lazy answer, reaching for
a port rule "just for this one app," is how a next-gen firewall quietly degrades into a legacy
one, one exception at a time.

## Why an interviewer cares

Anyone can enable App-ID. Knowing that `application-default` is what makes it airtight, and that
the failure mode is breaking a legitimate non-standard-port app rather than letting an attack
through, is the difference between having clicked the checkbox and understanding the control.
That understanding is what this lab is built to demonstrate.
