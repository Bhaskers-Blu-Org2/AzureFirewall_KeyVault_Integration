# The Challenge – Private Endpoints for Azure Key Vault

Currently, Azure Key Vault offers only public IP endpoints for device, client, and app connectivity.  While all communication with Azure Key Vault requires an encrypted TLS/SSL channel, there are customers who prefer device communication with Key Vault to occur over a private connection.  

There are several important use cases where Azure Key Vault would benefit from offering a private
endpoint to devices, clients, and apps:

- Private traffic though ExpressRoute (e.g., factory devices with secure private IPs that use MPLS for Cloud connectivity)
- You are using Key Vault to store encryption keys, application secrets, and certificates, and you want to block access to your key vault from the public internet
- You have an application running in your Azure virtual network, and this virtual network is locked down for all inbound and outbound traffic. Your application still needs to connect to Key Vault to fetch secrets and certificates or use cryptographic keys.

# The Solution – Azure Firewall as a Private Key Vault Gateway

Azure Firewall is a managed, cloud-based network security service which provides fully stateful firewall inspection with built-in high-availability and unrestricted cloud scalability.

Its applicability to solving this private gateway challenge is as follows:

- Azure Firewall is used as an HA scale-out tier that provides a private IP endpoint for Azure Key Vault clients, devices, and apps.
- Azure Firewall can provide a single endpoint for multiple Key Vaults while providing granular control with full auditing capabilities
- Azure Firewall can leverage service endpoints to prevent storage accessibility from any other network while allowing accessibility to on-prem resources. 

# The Essential Architecture

Virtual Network (VNet) Service Endpoints extends your virtual network private address space, and
the identity of your VNet, to multi-tenant Azure PaaS services over a special NAT tunnel in the
Azure fabric.  Service Endpoints allow you to secure your critical Azure service resources to only
your virtual networks. Traffic from your VNet to the Azure PaaS service always remains on the
Microsoft Azure backbone network.

The key here is the ability to deploy Azure Key Vault with Service Endpoints, while also making these
private PaaS endpoints available over an IPSEC VPN or the ExpressRoute private peering. The current limitation of Service Endpoints is that it is not accessible to on-premise resources.  Using service endpoints with Azure Firewall gives on-premise devices a private endpoint to hit coupled with a stateful security device providing centralized logging.  

The following diagram illustrates on-premise to cloud communication architecture secured via the
Azure Firewall hosted in Microsoft Azure. 

![alt text](https://github.com/microsoft/AzureFirewall_KeyVault_Integration/blob/master/images/topology.PNG) 

# Lab Components

**Resource group for HUB VNet and a minimum of two subnets:**
- GatewaySubnet (contains Azure ExpressRoute Gateway, /27 min, /26 recommended)
- AzureFirewallSubnet (contains Azure Firewall and will scale based on subnet size, minimum /26)
- Service Endpoints enabled for Microsoft Key Vault
- VNET Peering to Department 1 VNET

**Resource group for Department 1 VNet and a minimum of one subnet:**
- Server Subnet
- Test VM (used for testing access to Dept 1 Key Vault)
- VNET Peering to HUB VNET

**On-prem connectivity and test host:**
- ExpressRoute private peering or IPSEC VPN 
- Test host (used for testing access to On-Prem Key Vault)

**Key Vaults with access limited to AzureFirewallSubnet:**
- On-Prem Vault with a unique secret
- Dept1 Vault with a unique secret

DNS Manipulation

When the Key Vaults are created, a unique FQDN will be automatically generated based upon the name of the resource.  For example, when we create a Key Vault named dept1 and we look at the properties of that resource we will see a DNS Name FQDN of https://dept1.vault.azure.net/ as seen below.

![alt text](https://github.com/microsoft/AzureFirewall_KeyVault_Integration/blob/master/images/dns.PNG) 

As this FQDN will naturally resolve to a public IP address, we need to modify DNS entries in order to redirect traffic to the private endpoint of the Azure Firewall.  We can do this with a single host by modifying the local hosts file on the machine or, for multiple hosts, by creating an authoritative record for this FQDN within our local domain.

We must ensure that we do not modify the FQDN as this would break TLS and give users a certificate warning.  Future support for importing of custom certificates is coming to Azure Firewall however, this functionality does not exist today.  

# Network Configuration

**Azure Firewall:**

The Azure Firewall Application Rule Collection is configured to allow access from appropriate resources to their respective Key Vault FQDNs as seen below.

![alt text](https://github.com/microsoft/AzureFirewall_KeyVault_Integration/blob/master/images/portal1.PNG) 

As the Azure Firewall is a default deny device, any traffic destined to these endpoints which is not explicitly defined in these rulesets will be dropped.  This ensures that no unapproved endpoints obtain access to the Key Vaults as well as prevents data exfiltration between Vaults.

**Azure Key Vault:**

To ensure the Key Vaults can only be accessible via the Azure Firewall, we enabled service endpoints on the AzureFirewallSubnet and locked all Key Vault instances down to only this subnet.  To enforce this, we only allow access from the AzureFirewallSubnet within the Firewall and virtual networks blade of each Key Vault as seen below.

![alt text](https://github.com/microsoft/AzureFirewall_KeyVault_Integration/blob/master/images/portal2.PNG)

**Endpoint Devices:**

As mentioned in the DNS Manipulation section of this document, we must ensure traffic destined to each endpoint’s respective Vault is sent to the Azure Firewall.  For the Department 1 host within Azure we modified the hosts file locally.  For on-premise devices, we created an authoritative record within DNS as modifying the hosts file on all machines was impractical.

# Testing and Validation

**Testing Connectivity:**
To validate this setup, we installed PowerShell with Azure modules https://docs.microsoft.com/en-us/powershell/azure/install-az-ps?view=azps-2.0.0) on both the Department 1 and on-prem hosts and attempted to access the Key Vaults.  As displayed in the screenshots below, the on-prem host was able to reach its respective Vault and retrieve the keys while receiving an error message when accessing the department 1 Vault and vice versa.

![alt text](https://github.com/microsoft/AzureFirewall_KeyVault_Integration/blob/master/images/test1.PNG)

![alt text](https://github.com/microsoft/AzureFirewall_KeyVault_Integration/blob/master/images/test2.PNG)
 


Step by Step configuration of this lab can be found here: TBD
This lab can be deployed via a template here: TBD
