# Firewall and Network Segmentation Policy

## 1. Purpose

This document describes the network segmentation and firewall policy implemented on ROUTER01 using nftables.

The objective is to enforce least-privilege communication between the Management, Web, Application, and Data network segments while preserving required administrative access and Internet connectivity.

The firewall follows a **default-deny** approach for forwarded traffic.

---

## 2. Security Zones

The environment is divided into four internal security zones.

| Security Zone | Subnet | Host | Host IP |
|---|---|---|---|
| Management | 10.10.10.0/24 | ADMIN01 | 10.10.10.10 |
| Web | 10.10.20.0/24 | WEB01 | 10.10.20.10 |
| Application | 10.10.30.0/24 | APP01 | 10.10.30.10 |
| Data | 10.10.40.0/24 | DATA01 | 10.10.40.10 |

Each zone is implemented as a separate VirtualBox Internal Network.

Traffic between these networks must traverse ROUTER01.

ROUTER01 therefore acts as the centralized Layer 3 routing and firewall enforcement point.

---

## 3. Default-Deny Policy

The nftables forwarding chain uses a default-drop policy:

```nft
chain forward {
    type filter hook forward priority filter;
    policy drop;
}
```

Any forwarded traffic that does not match an explicit allow rule is dropped.

The security model is therefore:

```text
DENY BY DEFAULT
       +
EXPLICITLY ALLOW REQUIRED FLOWS
```

This reduces unnecessary communication paths between network zones.

---

## 4. Stateful Connection Tracking

The firewall permits return traffic associated with authorized connections using:

```nft
ct state established,related accept
```

This rule is required because the forwarding policy is `drop`.

For example:

```text
ADMIN01                           WEB01
10.10.10.10                  10.10.20.10
     │                            │
     │──── TCP/22 ───────────────>│
     │                            │
     │<──── Return traffic ───────│
     │    ESTABLISHED             │
```

Only the initial connection must match the appropriate explicit allow rule. Return traffic belonging to that connection is accepted through connection tracking.

---

## 5. Administrative Access

ADMIN01 is the dedicated management host.

SSH administration from ADMIN01 is explicitly permitted to the Web, Application, and Data servers:

```nft
ip saddr 10.10.10.10 ip daddr 10.10.20.10 tcp dport 22 accept
ip saddr 10.10.10.10 ip daddr 10.10.30.10 tcp dport 22 accept
ip saddr 10.10.10.10 ip daddr 10.10.40.10 tcp dport 22 accept
```

This produces the following authorized management flows:

| Source | Destination | Protocol | Port | Action |
|---|---|---|---:|---|
| ADMIN01 | WEB01 | TCP | 22 | Allow |
| ADMIN01 | APP01 | TCP | 22 | Allow |
| ADMIN01 | DATA01 | TCP | 22 | Allow |

SSH key-based authentication was configured and successfully tested for these administrative connections.

---

## 6. Web-to-Application Traffic

WEB01 is permitted to initiate TCP connections to APP01 on port 8080:

```nft
ip saddr 10.10.20.10 ip daddr 10.10.30.10 tcp dport 8080 accept
```

Authorized application path:

```text
WEB01
10.10.20.10
     │
     │ TCP/8080
     ▼
APP01
10.10.30.10
```

TCP port 8080 was used as a controlled application connectivity test.

Other new WEB01-to-APP01 connections are denied unless another explicit rule permits them.

For example, SSH from WEB01 to APP01 on TCP port 22 was tested and blocked.

---

## 7. Application-to-Data Traffic

APP01 is permitted to initiate connections to DATA01 on TCP port 5432:

```nft
ip saddr 10.10.30.10 ip daddr 10.10.40.10 tcp dport 5432 accept
```

Authorized path:

```text
APP01
10.10.30.10
     │
     │ TCP/5432
     ▼
DATA01
10.10.40.10
```

Port 5432 represents the database communication path in the three-tier architecture.

During validation, a temporary Python HTTP server was bound to TCP port 5432 to test TCP reachability.

No PostgreSQL database was deployed during this specific connectivity test.

---

## 8. Web-to-Data Isolation

No firewall rule permits WEB01 to initiate connections directly to DATA01.

Therefore the intended architecture is:

```text
WEB01 ────────> APP01 ────────> DATA01
        8080             5432
```

and not:

```text
WEB01 ───────────── X ─────────────> DATA01
```

A connection from:

```text
10.10.20.10 → 10.10.40.10:5432
```

was tested and successfully blocked.

This demonstrates isolation between the Web and Data tiers.

---

## 9. Internet Egress

The current firewall permits the internal laboratory address space to initiate outbound connections through the WAN interface:

```nft
ip saddr 10.10.0.0/16 oifname "enp0s3" accept
```

ROUTER01 then performs source NAT:

```nft
ip saddr 10.10.0.0/16 oifname "enp0s3" masquerade
```

This provides Internet connectivity for the internal virtual machines.

Internet egress filtering is intentionally broad during this phase so that inter-zone segmentation can be validated independently.

More restrictive outbound filtering is a future hardening opportunity.

---

## 10. Validated Traffic Matrix

The following TCP flows were tested after activating the default-deny policy:

| Source | Destination | Protocol / Port | Expected | Observed |
|---|---|---|---|---|
| ADMIN01 | WEB01 | TCP 22 | Allow | Allow |
| ADMIN01 | APP01 | TCP 22 | Allow | Allow |
| ADMIN01 | DATA01 | TCP 22 | Allow | Allow |
| WEB01 | APP01 | TCP 8080 | Allow | Allow |
| WEB01 | APP01 | TCP 22 | Deny | Deny |
| APP01 | DATA01 | TCP 5432 | Allow | Allow |
| WEB01 | DATA01 | TCP 5432 | Deny | Deny |
| DATA01 | APP01 | TCP 8080 | Deny | Deny |

All targeted TCP segmentation tests produced the expected results.

---

## 11. ICMP Behavior

No explicit inter-zone ICMP allow rules were added.

Therefore inter-zone ping traffic is blocked by the default-drop forwarding policy.

For example:

```text
WEB01 → APP01     ICMP     DROP
WEB01 → DATA01    ICMP     DROP
```

These results were observed during testing.

ICMP blocking is not the primary security objective of the lab. TCP-specific connectivity tests were used to validate the actual application segmentation policy.

---

## 12. Firewall Persistence

The nftables service was verified as:

```text
enabled
active
```

ROUTER01 was then restarted.

After reboot, the following controls were successfully validated:

- nftables remained active;
- the forwarding policy remained `drop`;
- `ct state established,related accept` remained active;
- NAT masquerading remained configured;
- IPv4 forwarding remained enabled;
- ADMIN01 could still establish authorized SSH connections;
- WEB01 retained Internet connectivity;
- DNS resolution continued to work.

This confirmed that the firewall configuration persisted across a system restart.

---

## 13. Security Principles Demonstrated

### Network Segmentation

Management, Web, Application, and Data workloads are placed in separate network segments.

### Default Deny

Forwarded traffic is denied unless explicitly authorized.

### Least Privilege

Only required inter-zone communication paths are permitted.

### Dedicated Management Plane

Administrative SSH access originates from ADMIN01 rather than from application workloads.

### Stateful Filtering

Connection tracking permits legitimate response traffic without broadly opening reverse communication paths.

### Defense in Depth

Network segmentation complements SSH key-based authentication and workload-level security controls.

---

## 14. Azure Relevance

This laboratory develops concepts relevant to Azure network security, including:

- VNet and subnet planning;
- network segmentation;
- least-privilege traffic rules;
- source and destination filtering;
- protocol and port filtering;
- stateful firewall concepts;
- management network separation;
- multi-tier application architecture;
- routing and Network Virtual Appliance concepts.

The local implementation is **not a direct replacement for Azure Network Security Groups**.

nftables performs centralized filtering on ROUTER01, while Azure NSGs are Azure-managed stateful packet filters that can be associated with subnets and network interfaces.

An Azure-native implementation would require separate NSG, routing, and outbound-connectivity configuration.

---

## 15. Future Hardening

Potential improvements include:

- Restrict Internet egress by zone and required service.
- Add logging for selected dropped connections.
- Add host-based firewall controls.
- Deploy actual Web, Application, and Database services.
- Centralize security and network logs.
- Define equivalent Azure NSG rules.
- Implement the Azure architecture using Bicep.
- Validate the design in an Azure subscription when available.