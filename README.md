# Azure Cloud Security Portfolio

## Overview

This repository is a hands-on portfolio focused on **Microsoft Azure administration, infrastructure, automation, and cloud security**.

The objective is to demonstrate practical skills aligned with the **AZ-104 Azure Administrator** skill set while progressively extending the environment toward **Cloud Security Analyst / Cloud Security Engineer** competencies.

Because the portfolio is developed without an active Azure subscription, the project combines:

* Local hands-on laboratories using virtual machines and Linux
* Azure architecture design
* Azure Infrastructure as Code using Bicep
* Azure CLI and PowerShell administration
* Networking and security simulations
* Technical documentation
* Troubleshooting scenarios
* Evidence of completed laboratory work

The portfolio clearly distinguishes between locally executed labs and Azure-specific implementations that require an Azure subscription.

---

## Lab Environment

The laboratory runs on a local Windows workstation capable of hosting multiple virtual machines and isolated virtual networks.


### Administration & Development Tools

* PowerShell 7
* Azure CLI
* Bicep CLI
* Git
* GitHub
* Visual Studio Code
* WSL2
* Ubuntu 24.04
* VirtualBox

Additional tools will be introduced as the portfolio progresses.

---

## Portfolio Architecture

The portfolio follows a progressive lab structure.

```text
azure-cloud-security-portfolio/
│
├── 00-workstation-setup/
├── 01-identity-governance/
├── 02-networking/
├── 03-storage/
├── 04-compute/
├── 05-monitoring-kql/
├── 06-automation/
├── 07-bicep-iac/
├── 08-cloud-security/
│
├── capstone-secure-azure-environment/
│
└── diagrams/
```

Each lab may contain:

```text
LAB/
│
├── README.md
├── architecture/
├── local-lab/
├── azure-equivalent/
├── bicep/
├── scripts/
└── evidence/
```

---

## Learning Path

| Lab      | Area                      | Key Technologies                                                |
| -------- | ------------------------- | --------------------------------------------------------------- |
| LAB 00   | Workstation Setup         | PowerShell, Git, VS Code, WSL2, Azure CLI, Bicep                |
| LAB 01   | Identity & Governance     | Entra ID, RBAC, Azure Policy, Resource Governance               |
| LAB 02   | Networking                | VNet, Subnets, NSG, Routing, DNS, NAT                           |
| LAB 03   | Storage                   | Storage Accounts, Encryption, Access Control, Private Access    |
| LAB 04   | Compute                   | Virtual Machines, Disks, Availability, Containers               |
| LAB 05   | Monitoring                | Azure Monitor, Log Analytics, KQL, Alerts                       |
| LAB 06   | Automation                | PowerShell, Bash, Azure CLI                                     |
| LAB 07   | Infrastructure as Code    | Bicep, ARM concepts, modular deployments                        |
| LAB 08   | Cloud Security            | Network Security, Identity Security, Least Privilege, Hardening |
| Capstone | Secure Azure Architecture | Integration of administration and security concepts             |

---

## Local Lab vs Azure

The local environment is used to reproduce the **technical principles behind Azure services**.

For example:

| Local Laboratory           | Azure Concept                                    |
| -------------------------- | ------------------------------------------------ |
| VirtualBox virtual network | Azure Virtual Network                            |
| Network segmentation       | Azure Subnets                                    |
| Linux/Windows VM           | Azure Virtual Machine                            |
| Linux firewall             | Network Security Group / Azure Firewall concepts |
| Linux routing              | Azure Route Tables / UDR concepts                |
| NAT                        | Azure NAT Gateway concepts                       |
| Local DNS                  | Azure DNS / Private DNS concepts                 |
| Virtual disks              | Azure Managed Disks concepts                     |
| VM snapshots               | Azure Snapshot concepts                          |
| Nginx / HAProxy            | Azure Load Balancer concepts                     |
| Docker                     | Azure container concepts                         |
| Local monitoring           | Azure Monitor concepts                           |
| PowerShell / Bash          | Azure administration automation                  |

These local implementations are **not presented as actual Azure deployments**.

Instead, each relevant lab documents how the locally implemented concept maps to its Azure equivalent.

---

## Infrastructure as Code

Azure infrastructure will progressively be represented using **Bicep**.

Example architecture:

```text
main.bicep
│
└── modules/
    ├── network.bicep
    ├── nsg.bicep
    ├── storage.bicep
    ├── compute.bicep
    └── monitoring.bicep
```

The objective is to demonstrate the ability to translate architecture requirements into reusable Infrastructure as Code.

---

## Security Approach

Security is integrated throughout the portfolio rather than treated as a separate final step.

The labs progressively introduce concepts such as:

* Least privilege
* Network segmentation
* Defense in depth
* Secure administrative access
* Identity-based access
* RBAC
* Network filtering
* Private connectivity
* Encryption
* Secrets management concepts
* Logging and monitoring
* Security hardening
* Infrastructure as Code validation
* Troubleshooting and remediation

---

## Troubleshooting Methodology

Labs will include intentionally introduced configuration problems.

The troubleshooting process will be documented using the following methodology:

```text
Problem
   ↓
Investigation
   ↓
Evidence Collection
   ↓
Root Cause
   ↓
Remediation
   ↓
Validation
   ↓
Azure Equivalent
```

This provides evidence not only of configuration skills but also of operational troubleshooting and security analysis.

---

## Evidence

Completed labs include technical evidence where appropriate.

Examples include:

* Command outputs
* Configuration files
* Network connectivity tests
* Firewall validation
* Routing tests
* Screenshots
* Logs
* Bicep validation results
* Troubleshooting notes

Sensitive information, credentials, private keys, tokens, and secrets are excluded from the repository.

---

## Skills Demonstrated

This portfolio is designed to progressively demonstrate competencies in:

**Azure Administration**

* Azure resource architecture
* Identity and governance
* Compute
* Storage
* Networking
* Monitoring

**Cloud Security**

* Identity security
* Network security
* Least privilege
* Security hardening
* Secure architecture
* Logging and monitoring
* Security posture concepts

**Infrastructure as Code**

* Bicep
* Modular infrastructure design
* Configuration validation
* Reusable infrastructure components

**Automation**

* PowerShell
* Bash
* Azure CLI

**Systems**

* Windows administration
* Linux administration
* WSL2
* Virtual machines
* Virtual networking

**DevOps Foundations**

* Git
* GitHub
* Version control
* Technical documentation

---

## Current Progress

### Completed

* [x] Local workstation preparation
* [x] PowerShell 7 installation
* [x] Git installation and configuration
* [x] Visual Studio Code installation
* [x] Azure CLI installation
* [x] Bicep CLI installation
* [x] WSL2 configuration
* [x] Ubuntu 24.04 installation
* [x] Git repository initialization
* [x] GitHub repository integration

### In Progress

* [ ] VirtualBox laboratory configuration
* [ ] Virtual network architecture
* [ ] First Linux virtual machines

### Upcoming

* [ ] Identity & Governance
* [ ] Azure Networking
* [ ] Storage
* [ ] Compute
* [ ] Monitoring & KQL
* [ ] Automation
* [ ] Bicep Infrastructure as Code
* [ ] Cloud Security labs
* [ ] Secure Azure Architecture Capstone

---

## Repository Status

This portfolio is under active development.

Each lab is added progressively with its architecture, implementation, validation, evidence, troubleshooting documentation, and Azure mapping.
