# Azure Secure 2-Tier Web Application Lab

## Overview

This project demonstrates the design and deployment of a secure two-tier web application architecture in Microsoft Azure.

The lab uses a public web tier and a private backend tier to show how Azure networking, subnet segmentation, Network Security Groups, and private VM-to-VM communication can be used to reduce public exposure and control traffic flow.

The environment was deployed manually through the Azure portal and validated using SSH, TCP connectivity testing, and ICMP testing.

## Skills Demonstrated

- Azure Virtual Network design
- CIDR subnetting and address planning
- Public and private subnet segmentation
- Network Security Group configuration
- NSG rule priority and traffic control
- Secure SSH administration
- Linux virtual machine administration
- Private VM-to-VM connectivity
- TCP connectivity validation with Netcat
- ICMP connectivity testing
- Cloud infrastructure documentation
- Security-first Azure architecture design

## Business Scenario

A small organization needs to host a web application in Azure while keeping backend resources protected from direct internet exposure.

The design separates the public-facing web server from the private backend tier. Only the web VM receives a public IP address, while the backend VM remains accessible only through the private Azure network.

This approach reduces the attack surface and demonstrates a basic secure cloud architecture pattern used in real environments.


## Architecture

```mermaid
flowchart LR
    User["Admin Workstation"] -->|SSH 22| PIP["Web VM Public IP"]
    Internet["Internet"] -->|HTTP 80| PIP
    PIP --> WEB["vm-web-01<br/>10.0.1.4"]
    WEB -->|Private VNet traffic| BACKEND["Backend VM<br/>vm-db-01<br/>10.0.2.4"]

    subgraph VNET["vnet-2tier-web<br/>10.0.0.0/16"]
        subgraph WEBNET["snet-web<br/>10.0.1.0/24"]
            WEB
        end

        subgraph BACKENDNET["snet-db<br/>10.0.2.0/24"]
            BACKEND
        end
    end
```
 
The web VM has a public IP for controlled administrative and web access. The backend VM has **no public IP** and is reachable only through private Azure networking.

## Azure Resources

| Resource | Name / Configuration |
|---|---|
| Resource Group | `rg-2tier-web-lab` |
| Region | `East US` |
| Virtual Network | `vnet-2tier-web` |
| VNet Address Space | `10.0.0.0/16` |
| Web Subnet | `snet-web` — `10.0.1.0/24` |
| Backend Subnet | `snet-db` — `10.0.2.0/24` |
| Web VM | `vm-web-01` — Ubuntu 24.04 LTS |
| Web Private IP | `10.0.1.4` |
| Web NSG | `vm-web-01-nsg` |
| Backend VM | `vm-db-01` — Ubuntu 24.04 LTS |
| Backend Private IP | `10.0.2.4` |
| Backend NSG | `vm-db-01-nsg` |
| Backend Public IP | None |


## Security Design

The lab uses network segmentation and least-privilege NSG rules to separate the public-facing web tier from the private backend tier.

### Web Tier NSG

The web VM NSG permits:

- TCP/80 from the Internet for HTTP traffic.
- TCP/22 only from the administrator's approved public IP for SSH management.
- Azure default NSG rules remain in place below the custom rules.

![Web NSG rules](screenshots/04-web-nsg-rules.png)

### Backend Tier NSG

The backend VM NSG is more restrictive:

- TCP/3306 is allowed from `10.0.1.0/24` to represent simulated application-to-backend/database traffic.
- TCP/22 is allowed from `10.0.1.0/24` for private administrative connectivity through the web tier.
- Other VNet inbound traffic is explicitly denied by a higher-priority custom deny rule before Azure's default `AllowVNetInBound` rule.
- The backend VM does not have a public IP address.

![Backend NSG rules](screenshots/06-db-nsg-rules.png)

## Network Security Group Rules

### Web Tier NSG

| Source | Destination | Port | Protocol | Action | Purpose |
|---|---|---|---|---|---|
| Admin public IP | Web VM | 22 | TCP | Allow | Restricted SSH administration |
| Internet | Web VM | 80 | TCP | Allow | HTTP web access |
| Azure defaults | Azure defaults | Various | Various | Default | Standard Azure NSG behavior |

### Backend Tier NSG

| Source | Destination | Port | Protocol | Action | Purpose |
|---|---|---|---|---|---|
| `10.0.1.0/24` | Backend VM | 3306 | TCP | Allow | Simulated application-to-backend/database traffic |
| `10.0.1.0/24` | Backend VM | 22 | TCP | Allow | Private SSH administration through web tier |
| `10.0.0.0/16` | Backend VM | Any | Any | Deny | Blocks broad VNet inbound traffic before default allow rule |

## Deployment Summary

The project was built manually in the Azure portal using the following high-level sequence:

1. Created the `rg-2tier-web-lab` resource group in East US.
2. Created `vnet-2tier-web` with address space `10.0.0.0/16`.
3. Created two subnets: `snet-web` (`10.0.1.0/24`) and `snet-db` (`10.0.2.0/24`).
4. Deployed `vm-web-01` into the web subnet with a public IP.
5. Configured the web NSG for HTTP and restricted SSH access.
6. Deployed `vm-db-01` into the backend subnet without a public IP.
7. Configured the backend NSG to permit required traffic from the web subnet while denying other VNet inbound traffic.
8. Connected to the web VM using SSH.
9. Validated private connectivity from the web VM to the backend VM.

## Validation

### SSH Access to the Web VM

SSH access to `vm-web-01` was successfully established from the administrator workstation using an SSH key.

![SSH connection to web VM](screenshots/07-ssh-web-vm.png)

### Private TCP Connectivity

From `vm-web-01`, TCP connectivity to the private backend VM on port 22 succeeded:

```bash
nc -vz -w 5 10.0.2.4 22
```

Result:

```text
Connection to 10.0.2.4 22 port [tcp/ssh] succeeded!
```

![Private SSH connectivity](screenshots/08-private-ssh-connectivity.png)

### Private ICMP Connectivity

Private subnet-to-subnet reachability was also validated with ICMP:

```bash
ping -c 4 10.0.2.4
```

The test returned four replies with **0% packet loss**.

![Private ping connectivity](screenshots/09-private-ping-connectivity.png)

## Screenshots

### Completed Resource Group

![Completed resource group](screenshots/01-final-resource-group.png)

### VNet and Subnets

![VNet subnets](screenshots/02-vnet-subnets.png)

### Web VM

![Web VM overview](screenshots/03-web-vm-overview.png)

### Backend VM

![Backend VM overview](screenshots/05-db-vm-overview.png)

## What This Lab Demonstrates

- Azure Virtual Network design
- CIDR subnetting and address planning
- Public and private VM architecture
- Network Security Group rule design and priority
- Restricting SSH management access
- Backend isolation by removing public exposure
- Private subnet-to-subnet communication
- Linux SSH administration
- TCP and ICMP connectivity testing
- Basic two-tier cloud security architecture

## Scope

This project focuses on **Azure networking and infrastructure security**. The backend VM represents a database tier, but installation and configuration of a MySQL/MariaDB database engine are intentionally outside the scope of this lab.

## Cleanup

After completing validation, remove the lab resources to avoid ongoing Azure charges.

Recommended cleanup steps:

1. Confirm that all screenshots and validation notes have been captured.
2. Delete the resource group `rg-2tier-web-lab`.
3. Verify that both VMs, disks, public IPs, NICs, and NSGs were removed.
4. Confirm that no unused public IPs or managed disks remain.


## Repository Structure

```text
azure-secure-2tier-web-app-lab/
├── README.md
├── docs/
│   └── architecture.md
└── screenshots/
    ├── 01-final-resource-group.png
    ├── 02-vnet-subnets.png
    ├── 03-web-vm-overview.png
    ├── 04-web-nsg-rules.png
    ├── 05-db-vm-overview.png
    ├── 06-db-nsg-rules.png
    ├── 07-ssh-web-vm.png
    ├── 08-private-ssh-connectivity.png
    └── 09-private-ping-connectivity.png
```

## Video Walkthrough

A short LinkedIn video walkthrough will be added after the lab is re-recorded.

The walkthrough will cover:

1. Architecture overview
2. Azure resource group
3. VNet and subnet segmentation
4. Web VM and backend VM design
5. NSG rule configuration
6. Private connectivity validation
7. Security lessons learned


## Key Takeaway

A secure two-tier design does not require every server to be internet-facing. By exposing only the web tier and keeping the backend tier on a private subnet, Azure NSGs can enforce controlled communication paths and reduce the attack surface.
