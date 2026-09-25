# Network Troubleshooting Case Studies

## 1. Purpose

This document records troubleshooting incidents encountered while building the segmented network laboratory.

Rather than documenting only the final working configuration, these case studies demonstrate the diagnostic process used to identify network and firewall problems, isolate their root causes, and validate corrective actions.

---

# Case Study 1 — DATA01 Could Not Reach Its Gateway

## 2. Problem

DATA01 was configured with the following network settings:

```text
IP address: 10.10.40.10/24
Default gateway: 10.10.40.1
Network: LAB-DATA
```

ROUTER01 had the corresponding interface:

```text
enp0s16: 10.10.40.1/24
```

Despite apparently correct Layer 3 configuration, DATA01 could not communicate with its default gateway.

---

## 3. Initial Hypotheses

Several possible causes were considered:

- incorrect IP address;
- incorrect subnet mask;
- incorrect default gateway;
- ROUTER01 interface down;
- incorrect routing configuration;
- firewall interference;
- VirtualBox network attachment problem.

The IP configuration and routing table were checked first.

They appeared correct.

---

## 4. ARP/Neighbor Investigation

The neighbor table on ROUTER01 showed the DATA01 address as incomplete.

Example:

```text
10.10.40.10 dev enp0s16 INCOMPLETE
```

An `INCOMPLETE` neighbor entry indicates that ROUTER01 attempted to resolve the Layer 2 address associated with 10.10.40.10 but did not receive a valid response.

This shifted the investigation from Layer 3 routing toward Layer 2 connectivity.

---

## 5. VirtualBox Investigation

The VirtualBox network configuration was inspected.

ROUTER01 was correctly connected to:

```text
LAB-DATA
```

through its fifth network adapter.

However, DATA01 was discovered to be connected to:

```text
LAB-APP
```

instead of:

```text
LAB-DATA
```

Therefore, although DATA01 had an IP address belonging to the 10.10.40.0/24 subnet, it was physically attached to the wrong VirtualBox Layer 2 network.

---

## 6. Root Cause

The root cause was a mismatch between the guest operating system's Layer 3 configuration and the VirtualBox Layer 2 network attachment.

```text
DATA01 IP configuration
10.10.40.10/24
        |
        | Expected
        v
LAB-DATA

Actual VirtualBox attachment
        |
        v
LAB-APP
```

Correct IP configuration alone was therefore insufficient to provide connectivity.

---

## 7. Resolution

The DATA01 VirtualBox network adapter was changed from:

```text
LAB-APP
```

to:

```text
LAB-DATA
```

After correcting the attachment, DATA01 successfully communicated with:

```text
10.10.40.1
```

and normal routing through ROUTER01 was restored.

---

## 8. Lessons Learned

This incident demonstrated the importance of troubleshooting networking layer by layer.

A useful diagnostic sequence is:

```text
IP configuration
      ↓
Routing table
      ↓
Gateway reachability
      ↓
ARP / Neighbor discovery
      ↓
Layer 2 connectivity
      ↓
Hypervisor network configuration
```

The `INCOMPLETE` neighbor entry was a key indicator that the problem existed below the IP routing layer.

---

# Case Study 2 — Valid nftables Configuration Blocked Legitimate Traffic

## 9. Problem

After changing the ROUTER01 forwarding policy from permissive filtering to:

```nft
policy drop;
```

previously working connections stopped functioning.

Symptoms included:

- SSH connection failures;
- Internet connectivity failures from internal systems;
- connection timeouts.

The firewall configuration had passed the nftables syntax validation command:

```bash
sudo nft -c -f /etc/nftables.conf
```

The command returned:

```text
0
```

This initially suggested that the configuration was valid.

---

## 10. Investigation

The firewall was intended to contain the following stateful rule:

```nft
ct state established,related accept
```

This rule allows return packets belonging to already authorized connections.

However, inspection of the active configuration revealed that the rule had accidentally been placed on the same line as a comment.

Conceptually, the configuration resembled:

```nft
# Allow return traffic ct state established,related accept
```

Everything after `#` was interpreted as a comment.

The connection-tracking rule therefore never became active.

---

## 11. Why Syntax Validation Did Not Detect the Problem

The configuration was syntactically valid.

From the nftables parser's perspective, the line was simply a valid comment.

Therefore:

```bash
sudo nft -c -f /etc/nftables.conf
```

returned successfully.

The problem was not a syntax error.

It was a **logical configuration error**.

This distinction is important:

```text
Syntax validation
      ≠
Security policy validation
```

A configuration can be syntactically correct while implementing the wrong security behavior.

---

## 12. Effect of the Missing Stateful Rule

Consider an authorized SSH connection:

```text
ADMIN01                         WEB01
     |                            |
     | ------ TCP SYN ----------> |
     |                            |
     | <----- TCP response ------ |
```

The initial ADMIN01-to-WEB01 packet matched the explicit TCP 22 allow rule.

However, without:

```nft
ct state established,related accept
```

the return traffic did not have a matching rule and encountered:

```nft
policy drop;
```

The connection therefore failed.

The same principle affected outbound Internet connections.

---

## 13. Recovery

The previously known-good firewall configuration was available as:

```text
/etc/nftables.conf.pre-segmentation
```

Loading the previous permissive configuration restored connectivity.

This confirmed that the failure was related to the new firewall policy rather than the underlying routing infrastructure.

The stateful rule was then corrected so that the comment and rule were on separate lines:

```nft
# Allow return traffic for authorized connections
ct state established,related accept
```

The corrected configuration was validated and activated.

---

## 14. Validation After Correction

The following capabilities were retested:

- ADMIN01 to WEB01 SSH;
- ADMIN01 to APP01 SSH;
- ADMIN01 to DATA01 SSH;
- Internet connectivity;
- DNS resolution;
- authorized application flows;
- unauthorized TCP flows.

The expected behavior was restored.

The firewall was subsequently tested after a ROUTER01 reboot to confirm persistence.

---

## 15. Lessons Learned

This incident demonstrated several security engineering principles.

### Validate behavior, not only syntax

Configuration validation tools detect structural errors but cannot guarantee that the implemented policy matches the intended security design.

### Maintain rollback configurations

The backup:

```text
/etc/nftables.conf.pre-segmentation
```

provided a fast recovery mechanism.

### Change security controls incrementally

Applying one major security change at a time makes failures easier to isolate.

### Test both positive and negative cases

A firewall should be tested for both:

```text
Traffic that SHOULD work
```

and:

```text
Traffic that SHOULD NOT work
```

Testing only successful connections does not demonstrate effective segmentation.

---

# 16. Troubleshooting Methodology

The laboratory reinforced the following general troubleshooting workflow:

```text
1. Observe the symptom
        ↓
2. Verify host configuration
        ↓
3. Verify Layer 2 connectivity
        ↓
4. Verify Layer 3 routing
        ↓
5. Inspect firewall state
        ↓
6. Compare intended vs active configuration
        ↓
7. Isolate one variable at a time
        ↓
8. Apply the correction
        ↓
9. Retest allowed traffic
        ↓
10. Retest denied traffic
        ↓
11. Validate persistence
```

This methodology helps distinguish infrastructure failures from security-policy failures.

---

# 17. Key Takeaways

The two incidents highlighted different classes of network problems:

| Incident | Layer / Domain | Root Cause | Diagnostic Indicator |
|---|---|---|---|
| DATA01 gateway failure | Layer 2 / Virtualization | Wrong VirtualBox Internal Network | ARP neighbor `INCOMPLETE` |
| Firewall connectivity failure | Stateful filtering | Connection-tracking rule commented out | Allowed connections timed out |

The first incident demonstrated that correct IP addressing does not guarantee Layer 2 connectivity.

The second demonstrated that syntactically valid security configuration does not guarantee logically correct security behavior.

Both incidents required verification of the actual system state rather than relying only on the intended configuration.