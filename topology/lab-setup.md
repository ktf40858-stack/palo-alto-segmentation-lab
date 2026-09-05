# Lab setup — PA-VM (VM-Series) trial

## What you need

| | |
|---|---|
| Firewall | Palo Alto PA-VM / VM-Series, 30-day trial from the Palo Alto support portal |
| Hypervisor | ESXi, KVM, or VMware Workstation — the PA-VM needs ~6 GB RAM, 4 vCPU |
| Endpoints | 4 small VMs: a user, a DMZ web host, an app host, a DB host |
| Cost | none for 30 days; the trial is the standard way to lab VM-Series |

Alternatives if the trial is not available: the Palo Alto **Cloud NGFW** free tier on AWS/Azure
marketplace, or a **PANW hosted lab** through the education portal. The `set` commands are
identical across all of them.

## Interface-to-zone map

| Interface | Zone | Subnet |
|---|---|---|
| ethernet1/1 | UNTRUST | 198.51.100.2/30 (documentation address) |
| ethernet1/2 | TRUST | 10.10.0.0/24 |
| ethernet1/3 | DMZ | 10.20.0.0/24 |
| ethernet1/4 | SERVERS | 10.30.0.0/24 |
| ethernet1/5 | MGMT | 10.99.0.0/24 |

## Deploy the policy

```
# SSH to the PA-VM, enter configure mode
configure
# paste policy/pan-os-set-commands.txt
commit
```

Then verify from the CLI:

```
# see the rule base
show running security-policy

# test a flow the way PAN-OS will evaluate it (this is the killer verification command)
test security-policy-match from TRUST to SERVERS source 10.10.0.20 \
     destination 10.30.0.20 protocol 6 destination-port 1433 application mssql-db
# -> should match "deny-all-log": a user host reaching the DB directly is denied
```

`test security-policy-match` is how you prove the design without generating traffic — it runs a
hypothetical packet through the actual rule base and tells you which rule catches it. Run it for
each scenario in [docs/03-east-west.md](../docs/03-east-west.md) and confirm the expected rule.

## Prove the segmentation

From the user VM, once endpoints are addressed:

```bash
# permitted: user -> app front-end on https  -> works
curl -k https://10.30.0.10

# denied: user -> database directly           -> times out, logged as deny
nc -vz 10.30.0.20 1433

# denied: user -> another server host          -> times out, logged as deny
ping 10.30.0.50
```

Each denied attempt appears in **Monitor > Traffic** with the session ended by `deny-all-log`.
That log entry — source, destination, application, the rule that denied it — is the deliverable:
it is what a SOC would see when the segmentation catches a real intrusion.
