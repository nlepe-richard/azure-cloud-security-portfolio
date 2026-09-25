# Network Segmentation and Security Lab

## Project Overview

This project demonstrates the design, implementation, and validation of a segmented network infrastructure using Oracle VirtualBox, Ubuntu Server, Linux routing, nftables, and SSH key-based administration.

The lab was built locally without an Azure subscription. It reproduces foundational networking and security concepts relevant to Azure Virtual Networks, subnet segmentation, network virtual appliances, and Network Security Groups.

**Important:** This is a local simulation of networking and security concepts, not a deployment of actual Azure resources.

## Objectives

* Design and deploy four isolated network segments: Management, Web, Application, and Data.
* Configure a Linux router to provide inter-subnet routing and Internet connectivity through NAT.
* Implement centralized, stateful firewall rules using nftables.
* Apply a default-deny policy to inter-zone traffic.
* Permit explicitly authorized administrative and application flows.
* Validate allowed and denied TCP connections.
* Verify firewall persistence after a router restart.
* Troubleshoot network connectivity and configuration issues.

## Lab Environment

| Component             | Technology                            |
| --------------------- | ------------------------------------- |
| Hypervisor            | Oracle VirtualBox                     |
| Operating system      | Ubuntu Server 26.04.1 LTS             |
| Router and firewall   | Linux, nftables                       |
| Network configuration | Netplan                               |
| Administration        | OpenSSH with key-based authentication |
| Connectivity testing  | ping, curl, SSH                       |
| Source control        | Git and GitHub                        |

## Network Architecture

The infrastructure consists of five virtual machines:

| Virtual Machine | Role                          | IP Address      | Network        |
| --------------- | ----------------------------- | --------------- | -------------- |
| ROUTER01        | Router, NAT gateway, firewall | 10.0.2.15 (WAN) | VirtualBox NAT |
| ADMIN01         | Administration workstation    | 10.10.10.10/24  | LAB-MGMT       |
| WEB01           | Web tier                      | 10.10.20.10/24  | LAB-WEB        |
| APP01           | Application tier              | 10.10.30.10/24  | LAB-APP        |
| DATA01          | Data tier                     | 10.10.40.10/24  | LAB-DATA       |

ROUTER01 connects all four internal networks and provides their default gateways:

* Management: 10.10.10.1/24
* Web: 10.10.20.1/24
* Application: 10.10.30.1/24
* Data: 10.10.40.1/24

Each internal network uses a separate VirtualBox Internal Network. Internal virtual machines have no direct VirtualBox NAT adapter.

## Security Implementation

The firewall uses a default-deny forwarding policy and explicitly permits selected traffic:

| Source  | Destination | Protocol / Port | Action |
| ------- | ----------- | --------------- | ------ |
| ADMIN01 | WEB01       | TCP 22          | Allow  |
| ADMIN01 | APP01       | TCP 22          | Allow  |
| ADMIN01 | DATA01      | TCP 22          | Allow  |
| WEB01   | APP01       | TCP 8080        | Allow  |
| APP01   | DATA01      | TCP 5432        | Allow  |
| WEB01   | DATA01      | TCP 5432        | Deny   |
| WEB01   | APP01       | TCP 22          | Deny   |
| DATA01  | APP01       | TCP 8080        | Deny   |

Established and related return traffic is permitted. The current policy also allows internal networks to initiate outbound connections through the WAN interface; Internet egress restrictions are outside the scope of this initial segmentation exercise.

## Validation Summary

All five targeted TCP segmentation tests produced the expected results:

* WEB01 to APP01 on TCP 8080: allowed.
* WEB01 to APP01 on TCP 22: blocked.
* APP01 to DATA01 on TCP 5432: allowed.
* WEB01 to DATA01 on TCP 5432: blocked.
* DATA01 to APP01 on TCP 8080: blocked.

TCP connectivity was tested using temporary Python HTTP servers. The service listening on TCP 5432 was a test HTTP server, not a PostgreSQL database.

Additional checks confirmed SSH key-based administration from ADMIN01, Internet connectivity and DNS resolution from WEB01, and firewall persistence after restarting ROUTER01.

## Troubleshooting

During implementation, DATA01 was initially attached to the wrong VirtualBox Internal Network (`LAB-APP` instead of `LAB-DATA`). The issue was identified through failed gateway connectivity, an incomplete ARP neighbor entry, and inspection of VirtualBox network adapter settings.

A second issue occurred when the nftables established/related connection-tracking rule was accidentally included in a comment. The configuration passed syntax validation but blocked legitimate return traffic. Separating the comment from the rule restored the intended behavior.

These incidents demonstrate a structured approach to network troubleshooting and security policy validation.

## Azure Relevance

This lab develops foundational knowledge relevant to:

* Azure Virtual Network address planning and subnet design.
* Routing and network virtual appliance concepts.
* Stateful traffic filtering and least-privilege network access.
* Network Security Group rule planning.
* Secure administrative access and network troubleshooting.

VirtualBox Internal Networks and Linux nftables are not direct replacements for Azure Virtual Networks or NSGs. Azure-specific implementations require separate validation in an Azure environment.

## Documentation

* [Network design](documentation/network-design.md)
* [Firewall policy](documentation/firewall-policy.md)
* [Validation results](documentation/validation-results.md)
* [Troubleshooting](documentation/troubleshooting.md)

## Status

Local network infrastructure, segmentation, connectivity testing, and post-reboot firewall validation completed. Azure deployment and infrastructure-as-code implementation remain future work.
