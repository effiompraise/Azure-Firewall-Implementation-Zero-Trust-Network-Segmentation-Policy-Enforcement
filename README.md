# Azure Firewall Deployment and Configuration: Enterprise Network Security Implementation

![Azure Firewall Banner](/images/Azure%20Firewall%20Banner.png)

## Introduction

In this project, I demonstrate the deployment and configuration of **Azure Firewall** as a cloud-native network security service, implementing a complete security perimeter for Azure workloads. This comprehensive implementation showcases the transition from an unprotected virtual network to a fully secured environment with centralized traffic inspection, application-level filtering, and secure remote access capabilities.

The project follows Microsoft's cloud security best practices and demonstrates practical implementation of defense-in-depth strategies, including network segmentation, traffic filtering, forced tunneling, and security policy enforcement. This architecture serves as a foundation for enterprise-grade cloud security implementations in production environments.

## Objectives

- Deploy Azure Firewall with forced tunneling support for enhanced security
- Configure Virtual Network infrastructure with proper subnet segmentation
- Implement application rules for FQDN-based filtering
- Establish network rules for protocol-level access control
- Configure DNAT rules for secure remote access to internal resources
- Create custom routing to force all traffic through the firewall
- Validate security policies through comprehensive testing

## Tech Stack

![Azure](https://img.shields.io/badge/Microsoft_Azure-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Azure Firewall](https://img.shields.io/badge/Azure_Firewall-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)
![Windows Server](https://img.shields.io/badge/Windows_Server_2022-0078D4?style=for-the-badge&logo=windows&logoColor=white)
![Networking](https://img.shields.io/badge/Azure_Networking-0089D6?style=for-the-badge&logo=microsoft-azure&logoColor=white)

- **Azure Firewall (Basic SKU)** - Cloud-native network security service
- **Azure Virtual Network** - Isolated network infrastructure
- **Azure Virtual Machine** - Windows Server 2022 for workload testing
- **Azure Route Tables** - Custom routing for traffic inspection
- **Firewall Policy** - Centralized rule management
- **Network Security** - DNS configuration and traffic control

## Architecture Overview

### Deployment Architecture
![Architecture Diagram](/images/Screenshot%202025-12-31%20052744.png)


The architecture implements a hub-and-spoke network topology with centralized security:

**Network Layer:**
- Virtual Network (10.0.0.0/16) with segmented subnets
- **AzureFirewallSubnet** (10.0.1.0/26) for firewall data plane
- **AzureFirewallManagementSubnet** (10.0.0.0/26) for forced tunneling
- **Workload-SN** (10.0.2.0/24) for application servers

**Security Layer:**
- Azure Firewall with dual public IPs (data and management)
- Firewall Policy with application, network, and DNAT rules
- Custom DNS servers for secure name resolution
- Route table forcing all traffic through firewall

**Traffic Flow:**
1. All outbound traffic from Workload-SN → Route Table → Azure Firewall → Internet
2. Inbound RDP traffic → Firewall Public IP → DNAT → VM Private IP
3. DNS queries → Firewall → Custom DNS servers (209.244.0.3, 209.244.0.4)

## Implementation Steps

### Phase 1: Virtual Network Infrastructure Deployment

Created a segmented virtual network to support Azure Firewall and workload isolation.

**Subnet Design:**

| Subnet Name | Address Range | Purpose |
|------------|---------------|---------|
| AzureFirewallSubnet | 10.0.1.0/26 | Firewall data plane |
| AzureFirewallManagementSubnet | 10.0.0.0/26 | Firewall management plane |
| Workload-SN | 10.0.2.0/24 | Application workloads |

![Virtual Network IP Configuration](/images/Virtual%20Network.png)
*Figure 1: Configuring Virtual Network IP address space and the Data plane subnet*

![Management Subnet Configuration](/images/Azure%20Firewall%20Subnet.png)
*Figure 2: Adding the dedicated Management Subnet for forced tunneling*

### Phase 2: Virtual Machine Deployment for Testing

Deployed a Windows Server VM in the workload subnet to test firewall policies and connectivity.

**VM Specifications:**
- **Name:** Srv-Workload
- **OS:** Windows Server 2022 Datacenter
- **Network:** Workload-SN (10.0.2.0/24)
- **Security:** Boot diagnostics disabled; No public IP assigned directly (access via DNAT).

![VM Deployment Complete](/images/VM%20deployment.png)
*Figure 3: Successful deployment of the workload VM*

### Phase 3: Azure Firewall and Policy Deployment

Deployed Azure Firewall with Basic SKU and created a centralized firewall policy for rule management.

**Firewall Configuration:**
- **Name:** MyFirewallTest
- **SKU:** Basic
- **Policy:** MyPolicy
- **Public IPs:** Separate IPs for Data (DNAT/SNAT) and Management (Forced Tunneling).

![Firewall Deployment](/images/Firewall%20Deployment.png)
*Figure 4: Azure Firewall resources successfully deployed*

### Phase 4: Custom Route Table Configuration

Created a route table to force all outbound traffic from the workload subnet through Azure Firewall for inspection.

**Route Configuration:**

| Route Name | Address Prefix | Next Hop Type | Next Hop Address |
|-----------|----------------|---------------|------------------|
| MyRoute | 0.0.0.0/0 | Virtual Appliance | 10.0.1.4 |

**How It Works:**
1. VM in Workload-SN initiates outbound connection.
2. Route table matches 0.0.0.0/0 (all internet traffic).
3. Traffic is redirected to the firewall's private IP (10.0.1.4) instead of going directly to the internet.

![Route Table Overview](/images/Route%20Table%20VA%20next%20hop.png)
*Figure 5: Route Table overview showing the 'Virtual Appliance' next hop*

![Route Association](/images/Route%20Table.png)
*Figure 6: Associating the Route Table with the Workload subnet*

### Phase 5: Application Rule Configuration

Implemented Layer 7 filtering to control which websites and applications workloads can access.

**Rule: Allow-Google**
- **Source:** 10.0.2.0/24
- **Protocol:** HTTP, HTTPS
- **Target FQDN:** `www.google.com`
- **Action:** Allow

**Security Benefit:** Enables granular FQDN filtering, ensuring workloads can only access approved domains (Whitelisting).

![Application Rule](/images/Application%20Rule.png)
*Figure 7: Configuring Application Rule to allow access to Google only*

### Phase 6: Network Rule Configuration

Configured Layer 4 filtering to allow DNS queries from the workload subnet to specific DNS servers (Quad9).

**Rule: Allow-DNS**
- **Protocol:** UDP
- **Port:** 53
- **Destination:** 209.244.0.3, 209.244.0.4

**Why This Matters:** Without this rule, the VM cannot resolve domain names to IP addresses, rendering the Application Rules useless.

![Network Rule](/images/Virtual%20Network.png)
*Figure 8: Configuring Network Rule to allow DNS traffic*

### Phase 7: DNAT Rule Configuration

Configured Destination Network Address Translation (DNAT) to enable secure RDP access to the VM through the firewall's public IP.

**Rule: rdp-nat**
- **Protocol:** TCP
- **Port:** 3389
- **Destination:** [Firewall Public IP]
- **Translated Address:** 10.0.2.x (VM Private IP)

**Security Benefit:** The VM does not need a public IP address. All access is proxied and logged through the firewall.

![DNAT Rule](/images/DNAT%20Rule.png)
*Figure 9: DNAT Rule configuration for secure RDP access*

### Phase 8: DNS Configuration on VM

Configured the VM's network interface to use the specific DNS servers allowed in Phase 6 (209.244.0.3, 209.244.0.4). This ensures DNS queries are routed correctly and match the firewall rules.

## Testing and Validation

### 1. RDP Access via DNAT
**Test:** Initiated RDP connection to the Firewall's Public IP.
**Result:** ✅ Success. Connection was forwarded to the internal VM.

### 2. Application Whitelisting
**Test:** Browsed to `www.google.com`.
**Result:** ✅ Success. Page loaded (matched Application Rule).

**Test:** Browsed to `www.microsoft.com`.
**Result:** ❌ Blocked. "Connection timed out" (Implicit Deny).

### 3. DNS Resolution
**Test:** `nslookup www.google.com`
**Result:** ✅ Success. Resolved via Quad9 servers through the firewall.

## Key Results

* **Security Controls:** 7 major security controls implemented (Segmentation, Firewall, Forced Tunneling, App Rules, Net Rules, DNAT, Custom DNS).
* **Zero Trust:** Achieved a "Deny by Default" posture where only explicitly allowed traffic is permitted.
* **Attack Surface Reduction:** Eliminated public IP exposure for the workload VM.
* **Traffic Visibility:** All inbound and outbound traffic is now inspected and logged by Azure Firewall.

## About This Project

**Role:** Cloud Security Engineer
**Skills Demonstrated:** Azure Networking, Infrastructure Security, Firewalls, Routing, Compliance.