# Network Architecture and Design

## 1. Overview

This document describes the network architecture of a local, segmented infrastructure built with Oracle VirtualBox and Ubuntu Server.

The environment consists of five virtual machines distributed across four isolated internal networks and one external NAT connection.

A Linux router, ROUTER01, provides inter-subnet routing, Internet connectivity through Network Address Translation (NAT), and centralized stateful traffic filtering with nftables.

The lab demonstrates foundational network architecture and security concepts relevant to cloud infrastructure administration.

**Scope:** This is a local VirtualBox implementation. It is not an Azure deployment.

## 2. Architecture Components

| Virtual Machine | Role                          | Operating System          |
| --------------- | ----------------------------- | ------------------------- |
| ROUTER01        | Router, NAT gateway, firewall | Ubuntu Server 26.04.1 LTS |
| ADMIN01         | Administrative workstation    | Ubuntu Server 26.04.1 LTS |
| WEB01           | Web-tier test server          | Ubuntu Server 26.04.1 LTS |
| APP01           | Application-tier test server  | Ubuntu Server 26.04.1 LTS |
| DATA01          | Data-tier test server         | Ubuntu Server 26.04.1 LTS |

All five virtual machines run on a local Oracle VirtualBox hypervisor.

## 3. IP Addressing Plan

The laboratory uses the private IPv4 address space 10.10.0.0/16 for its internal networks.

Four /24 subnets are allocated to separate infrastructure roles.

| Network     | Subnet        | Gateway    | Virtual Machine | VM IP Address |
| ----------- | ------------- | ---------- | --------------- | ------------- |
| Management  | 10.10.10.0/24 | 10.10.10.1 | ADMIN01         | 10.10.10.10   |
| Web         | 10.10.20.0/24 | 10.10.20.1 | WEB01           | 10.10.20.10   |
| Application | 10.10.30.0/24 | 10.10.30.1 | APP01           | 10.10.30.10   |
| Data        | 10.10.40.0/24 | 10.10.40.1 | DATA01          | 10.10.40.10   |

Each subnet is implemented as a separate VirtualBox Internal Network.

The 10.10.0.0/16 range is an address-planning boundary for the laboratory. The four /24 subnets are distinct Layer 2 networks connected through ROUTER01.

## 4. ROUTER01 Network Interfaces

ROUTER01 has five network interfaces.

| Interface | VirtualBox Attachment | IP Address    | Function              |
| --------- | --------------------- | ------------- | --------------------- |
| enp0s3    | NAT                   | 10.0.2.15/24  | External connectivity |
| enp0s8    | LAB-MGMT              | 10.10.10.1/24 | Management gateway    |
| enp0s9    | LAB-WEB               | 10.10.20.1/24 | Web gateway           |
| enp0s10   | LAB-APP               | 10.10.30.1/24 | Application gateway   |
| enp0s16   | LAB-DATA              | 10.10.40.1/24 | Data gateway          |

The external interface receives its IP configuration from VirtualBox NAT.

The four internal interfaces use static IPv4 addresses configured through Netplan.

## 5. Network Isolation

VirtualBox Internal Networks provide separate Layer 2 segments.

ADMIN01, WEB01, APP01, and DATA01 each connect to their designated internal network.

The internal virtual machines do not require individual VirtualBox NAT adapters. Their default routes point to the corresponding internal interface of ROUTER01.

Traffic between different internal subnets must therefore traverse ROUTER01, where the nftables forwarding policy determines whether a connection is permitted.

This design centralizes inter-zone filtering. It does not replace host-level firewall controls or provide traffic inspection between machines on the same Layer 2 segment.

## 6. IP Routing

IPv4 forwarding is enabled on ROUTER01:

```bash
net.ipv4.ip_forward = 1
```

The router maintains directly connected routes for the four internal subnets and a default route through its VirtualBox NAT interface.

Each internal virtual machine has a default route through its respective gateway.

For example, WEB01 uses:

```text
10.10.20.0/24 dev enp0s3
default via 10.10.20.1 dev enp0s3
```

This allows WEB01 to reach permitted destinations outside its local subnet through ROUTER01.

## 7. Internet Connectivity and NAT

ROUTER01 provides Internet access through its external interface, enp0s3.

Source NAT is implemented with nftables:

```nft
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        policy accept;

        ip saddr 10.10.0.0/16 oifname "enp0s3" masquerade
    }
}
```

The masquerade rule translates the source addresses of outbound packets from the internal networks to the router's external interface address.

VirtualBox NAT subsequently provides connectivity between the router's external interface and the host's external network.

The internal virtual machines do not have directly exposed public IP addresses.

## 8. Centralized Firewall Architecture

ROUTER01 uses nftables to control traffic forwarded between network segments.

The forwarding chain applies a default-deny policy:

```nft
chain forward {
    type filter hook forward priority filter;
    policy drop;

    ct state established,related accept

    # Additional explicit allow rules
}
```

New connections must match an explicit allow rule to pass through the router.

The connection-tracking rule permits return traffic associated with established or related connections.

The firewall policy permits administrative SSH access from ADMIN01 and selected application flows between the Web, Application, and Data tiers.

The current implementation also permits internal networks to initiate outbound traffic through the WAN interface. More restrictive Internet egress filtering is a potential future enhancement.

Detailed rules and permitted flows are documented in [Firewall Policy](firewall-policy.md).

## 9. Secure Administration

ADMIN01 serves as the dedicated administrative workstation.

SSH key-based authentication was configured and tested for administrative connections from ADMIN01 to WEB01, APP01, and DATA01.

The firewall explicitly permits these connections on TCP port 22.

Administrative SSH access from WEB01 to APP01 is not permitted by the inter-zone firewall policy.

The laboratory does not implement Microsoft Entra ID, Azure Bastion, or Azure Privileged Identity Management.

## 10. Validation

The following capabilities were validated:

* Connectivity between each internal virtual machine and its default gateway.
* IPv4 forwarding through ROUTER01.
* Internet access through NAT.
* DNS resolution from internal virtual machines.
* SSH key-based administration from ADMIN01.
* Authorized and denied inter-zone TCP connections.
* Firewall configuration persistence after restarting ROUTER01.

The detailed test matrix and observed outcomes are documented in [Validation Results](validation-results.md).

## 11. Azure Architecture Mapping

| Local Lab Component          | Related Azure Concept                                |
| ---------------------------- | ---------------------------------------------------- |
| Internal IP address planning | VNet address space and subnet planning               |
| VirtualBox Internal Networks | Conceptual model of isolated network segments        |
| ROUTER01                     | Network virtual appliance and routing concepts       |
| Linux forwarding and routing | IP forwarding and route design                       |
| nftables forwarding rules    | Stateful network filtering concepts relevant to NSGs |
| NAT on ROUTER01              | Outbound source NAT concepts                         |
| ADMIN01                      | Dedicated administrative access model                |

These mappings describe conceptual similarities rather than direct service equivalence.

Azure Virtual Networks, Network Security Groups, route tables, Azure NAT Gateway, and Azure Firewall have platform-specific capabilities and behavior that are not reproduced by VirtualBox and nftables.

A future phase of the portfolio will define Azure-native infrastructure using Bicep and validate the deployment when an Azure environment is available.

## 12. Future Improvements

Potential extensions include:

* Implementing a more restrictive Internet egress policy.
* Deploying an actual application and database service.
* Adding host-based firewall controls.
* Introducing network logging and traffic analysis.
* Creating an Azure-native network design and Bicep templates.
* Validating the Azure design in an actual Azure subscription.
