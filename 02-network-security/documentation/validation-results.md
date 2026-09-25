# Network Security Validation Results

## 1. Purpose

This document records the validation tests performed on the segmented network laboratory.

The objective was to verify that routing, NAT, DNS resolution, administrative access, stateful firewall filtering, network segmentation, and firewall persistence operated according to the intended design.

The tests were performed after implementing the nftables default-deny forwarding policy on ROUTER01.

---

## 2. Validation Environment

| Host | Role | IP Address |
|---|---|---|
| ADMIN01 | Management workstation | 10.10.10.10 |
| WEB01 | Web tier | 10.10.20.10 |
| APP01 | Application tier | 10.10.30.10 |
| DATA01 | Data tier | 10.10.40.10 |

ROUTER01 provides routing between the four networks and Internet connectivity through its VirtualBox NAT interface.

---

## 3. Routing Validation

Before applying restrictive firewall rules, connectivity between each virtual machine and its gateway was verified.

The following gateway addresses were tested:

| Network | Gateway |
|---|---|
| Management | 10.10.10.1 |
| Web | 10.10.20.1 |
| Application | 10.10.30.1 |
| Data | 10.10.40.1 |

The tests confirmed that all four internal networks were correctly connected to ROUTER01.

IPv4 forwarding was also verified on ROUTER01:

```bash
sysctl net.ipv4.ip_forward
```

Expected and observed state:

```text
net.ipv4.ip_forward = 1
```

---

## 4. NAT and Internet Connectivity

ROUTER01 performs source NAT for the internal address space:

```nft
ip saddr 10.10.0.0/16 oifname "enp0s3" masquerade
```

Internet connectivity was validated from internal hosts using:

```bash
ping -c 4 1.1.1.1
```

Successful responses confirmed IP connectivity through:

```text
Internal VM
    |
    v
ROUTER01
    |
    | nftables masquerade
    v
VirtualBox NAT
    |
    v
Internet
```

---

## 5. DNS Validation

DNS resolution was tested using:

```bash
getent ahostsv4 ubuntu.com
```

WEB01 successfully resolved the hostname after the restrictive firewall policy was activated.

This confirmed that Internet connectivity and DNS resolution remained operational after segmentation.

---

## 6. SSH Administrative Access

SSH key-based authentication was configured between ADMIN01 and the three workload servers.

The following administrative paths were validated:

| Source | Destination | Service | Result |
|---|---|---|---|
| ADMIN01 | WEB01 | SSH / TCP 22 | PASS |
| ADMIN01 | APP01 | SSH / TCP 22 | PASS |
| ADMIN01 | DATA01 | SSH / TCP 22 | PASS |

Example validation command:

```bash
ssh -i ~/.ssh/cloudlab_admin_ed25519 \
  -o IdentitiesOnly=yes \
  -o PasswordAuthentication=no \
  -o KbdInteractiveAuthentication=no \
  cloudadmin@10.10.20.10 hostname
```

Equivalent tests were performed against APP01 and DATA01.

The remote systems returned their expected hostnames:

```text
web01
app01
data01
```

This confirmed that administrative SSH remained available after the firewall forwarding policy was changed to default deny.

---

## 7. Default-Deny Validation

The ROUTER01 forwarding chain was configured with:

```nft
policy drop;
```

The active ruleset was verified using:

```bash
sudo nft list chain inet filter forward
```

The stateful return-traffic rule was also confirmed:

```nft
ct state established,related accept
```

This established a deny-by-default security posture for forwarded traffic.

---

## 8. TCP Segmentation Tests

Temporary Python HTTP servers were used to create deterministic TCP listeners for firewall testing.

On APP01:

```bash
python3 -m http.server 8080 --bind 10.10.30.10
```

On DATA01:

```bash
python3 -m http.server 5432 --bind 10.10.40.10
```

The service on TCP 5432 was only a TCP connectivity test. It was not a PostgreSQL database.

### Test Results

| Test | Source | Destination | Port | Expected | Observed | Status |
|---|---|---|---:|---|---|---|
| 1 | WEB01 | APP01 | TCP 8080 | Allow | Allowed | PASS |
| 2 | WEB01 | APP01 | TCP 22 | Deny | Blocked | PASS |
| 3 | APP01 | DATA01 | TCP 5432 | Allow | Allowed | PASS |
| 4 | WEB01 | DATA01 | TCP 5432 | Deny | Blocked | PASS |
| 5 | DATA01 | APP01 | TCP 8080 | Deny | Blocked | PASS |

**Result: 5/5 targeted TCP segmentation tests produced the expected outcome.**

---

## 9. WEB01 to APP01 — Allowed

The following command was executed from WEB01:

```bash
curl -4 --connect-timeout 5 --max-time 10 \
  -I http://10.10.30.10:8080/
```

The connection succeeded.

This validated the firewall rule permitting:

```text
WEB01
10.10.20.10
      |
      | TCP 8080
      v
APP01
10.10.30.10
```

Result: **PASS**

---

## 10. WEB01 to APP01 SSH — Denied

WEB01 attempted to establish an SSH connection to APP01 on TCP port 22.

The connection timed out as expected.

No firewall rule permits WEB01 to initiate SSH connections to APP01.

Result: **PASS**

---

## 11. APP01 to DATA01 — Allowed

APP01 tested connectivity to the temporary TCP service on DATA01:

```bash
curl -4 --connect-timeout 5 --max-time 10 \
  -I http://10.10.40.10:5432/
```

The connection succeeded.

This validated the intended application-to-data path:

```text
APP01
10.10.30.10
      |
      | TCP 5432
      v
DATA01
10.10.40.10
```

Result: **PASS**

---

## 12. WEB01 to DATA01 — Denied

WEB01 attempted to connect directly to DATA01 on TCP port 5432.

The connection timed out.

This confirmed that the Web tier cannot bypass the Application tier to access the Data tier directly.

```text
WEB01
10.10.20.10
      |
      | TCP 5432
      X
      |
DATA01
10.10.40.10
```

Result: **PASS**

---

## 13. DATA01 to APP01 — Denied

DATA01 attempted to initiate a TCP connection to APP01 on port 8080.

The connection was blocked by the default-deny forwarding policy.

This confirmed that permitting APP01-to-DATA01 traffic does not automatically authorize DATA01 to initiate new connections toward APP01.

Return traffic belonging to established connections remains permitted through connection tracking.

Result: **PASS**

---

## 14. ICMP Segmentation

Inter-zone ICMP traffic was not explicitly permitted.

Tests such as:

```bash
ping -c 4 10.10.30.10
ping -c 4 10.10.40.10
```

from WEB01 failed after the default-deny policy was activated.

This was expected.

TCP tests were used as the primary validation method because they directly represented the application flows defined in the firewall policy.

---

## 15. Firewall Persistence Test

The nftables service was checked before reboot:

```bash
sudo systemctl is-enabled nftables
sudo systemctl is-active nftables
```

Observed state:

```text
enabled
active
```

ROUTER01 was then restarted.

After reboot, the following controls were verified:

| Control | Result |
|---|---|
| nftables service active | PASS |
| Forward policy remains `drop` | PASS |
| `established,related` rule present | PASS |
| NAT masquerade rule present | PASS |
| `net.ipv4.ip_forward = 1` | PASS |
| ADMIN01 → WEB01 SSH | PASS |
| WEB01 Internet connectivity | PASS |
| WEB01 DNS resolution | PASS |

The firewall and routing configuration therefore survived the ROUTER01 reboot.

---

## 16. Overall Validation Result

The laboratory successfully demonstrated:

- Layer 3 routing between isolated network segments.
- Source NAT for outbound Internet connectivity.
- DNS connectivity.
- SSH key-based administration.
- Stateful packet filtering.
- Default-deny forwarding.
- Least-privilege inter-zone communication.
- Web/Application/Data tier isolation.
- Explicitly allowed application flows.
- Blocking of unauthorized TCP flows.
- Persistent firewall configuration after reboot.

### Final Result

**All planned network segmentation tests completed successfully.**

The environment is ready for documentation, architecture mapping, and future Infrastructure-as-Code work.