# VirtualBox Network Plan

## Address Space

Lab address space:

`10.10.0.0/16`

## Network Segments

| Network | CIDR | Purpose |
|---|---|---|
| Management | 10.10.10.0/24 | Administrative systems |
| Web | 10.10.20.0/24 | Web tier |
| Application | 10.10.30.0/24 | Application tier |
| Data | 10.10.40.0/24 | Data tier |

## Security Objective

The laboratory uses network segmentation to separate administrative, web, application, and data workloads.

Traffic between network segments will be explicitly controlled through routing and firewall rules.

## Azure Mapping

| Local Lab | Azure |
|---|---|
| Virtual network | VNet |
| Network segment | Subnet |
| Firewall rule | NSG rule |
| Router | Route Table / UDR concepts |
| NAT | NAT Gateway concepts |
| Firewall appliance | Azure Firewall concepts |