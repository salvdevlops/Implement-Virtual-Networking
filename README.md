# Implement Virtual Networking

This lab is about the fundamentals of virtual networking in Azure — building virtual networks and subnets, locking traffic down with network security groups and application security groups, and setting up public and private DNS zones.

## Overview

In this lab I built two virtual networks (one through the portal, one through an exported ARM template), set up an Application Security Group and Network Security Group to control traffic in and out of one of those networks, then configured both a public and a private DNS zone for name resolution.

## Lab Scenario

The org is rolling out virtual networks to cover its existing resources, with room to grow:

- **CoreServicesVnet** holds the largest number of resources and needs a big address space since most of the future growth lands here.
- **ManufacturingVnet** supports the manufacturing facilities, which are expected to bring on a large number of connected IoT-style devices over time.

Both networks and their subnets needed to be sized up front with that growth in mind — and ideally without any overlapping IP ranges between them, since that's just asking for routing headaches down the line.

## Environment

- Azure subscription

## Diagram

`[add diagram later]`

---

## Task 1: Create a Virtual Network with Subnets Using the Portal

First network was **CoreServicesVnet**, built straight through the portal since this one needed the larger address space for the org's core resources.

I searched for **Virtual Networks**, hit **Create**, and filled out the Basics:

| Setting | Value |
|---|---|
| Resource Group | az104-rg4 *(created new)* |
| Name | CoreServicesVnet |
| Region | (US) East US |

On the **Address space** tab, I replaced the default prepopulated range with `10.20.0.0/16`.

Then I added two subnets, deleting the default one Azure tries to create automatically:

| Subnet | Starting Address | Size |
|---|---|---|
| SharedServicesSubnet | 10.20.10.0 | /24 |
| DatabaseSubnet | 10.20.20.0 | /24 |

> Every virtual network needs at least one subnet, and worth remembering — Azure always reserves 5 IP addresses per subnet, so that eats into the usable range.

**Review + create**, confirmed it passed validation, then **Create**. Once it deployed I went to the resource and checked over the address space and subnets to make sure everything matched what I'd configured.

<img width="726" height="770" alt="image" src="https://github.com/user-attachments/assets/702108a7-9886-419f-b606-55be8b9a7d06" />

From the **Automation** section, I used **Export template** to generate a template based on this exact network, then downloaded both the Template and Parameters JSON files — I'd need these for the next task to recreate a similar setup for Manufacturing.

---

## Task 2: Create a Virtual Network and Subnets Using a Template

This time the goal was **ManufacturingVnet**, built off the template exported from CoreServicesVnet instead of clicking through the portal from scratch — same idea, different network, sized differently to fit what Manufacturing actually needs (room for a lot of connected devices).

Opened the `template.json` file from Task 1 in an editor and made these changes:

- Replaced every occurrence of `CoreServicesVnet` with `ManufacturingVnet`
- Replaced every occurrence of `10.20.0.0` with `10.30.0.0`
- Renamed `SharedServicesSubnet` to `SensorSubnet1`, and changed `10.20.10.0/24` to `10.30.20.0/24`
- Renamed `DatabaseSubnet` to `SensorSubnet2`, and changed `10.20.20.0/24` to `10.30.21.0/24`

Read back through the whole file afterward to double check nothing got missed, then saved.

I made the matching edit on `parameters.json` — just the one occurrence of `CoreServicesVnet` swapped to `ManufacturingVnet` — and saved that too.

To deploy: searched for **Deploy a custom template**, picked **Build your own template in the editor**, loaded the edited `template.json`, then loaded the edited `parameters.json` under **Edit parameters**. Confirmed the resource group was still `az104-rg4`, then **Review + create** → **Create**.

Once it finished deploying, I confirmed in the portal that **ManufacturingVnet** and its two subnets showed up correctly.

<img width="735" height="761" alt="image" src="https://github.com/user-attachments/assets/c1bff13e-62c2-4971-b13f-e8bbf8167ebf" />

---

## Task 3: Set Up an Application Security Group and Network Security Group

This task was about controlling traffic into and out of a subnet using an ASG (Application Security Group) paired with an NSG (Network Security Group) — letting traffic in from a specific application group while explicitly blocking outbound internet access.

**Creating the ASG:**

Searched for **Application security groups**, hit **Create**:

| Setting | Value |
|---|---|
| Subscription | my subscription |
| Resource group | az104-rg4 |
| Name | asg-web |
| Region | East US |

**Review + create** → **Create**.

<img width="801" height="341" alt="image" src="https://github.com/user-attachments/assets/75517b6c-4a6d-4d68-a94f-ab17f314bc22" />

> In a real setup, you'd attach this ASG to one or more VMs — those VMs would then be the ones actually affected by the inbound rule set up next.

**Creating the NSG and linking it to CoreServicesVnet:**

Searched for **Network security groups**, hit **+ Create**:

| Setting | Value |
|---|---|
| Subscription | my subscription |
| Resource group | az104-rg4 |
| Name | myNSGSecure |
| Region | East US |

**Review + create** → **Create**, then **Go to resource**.

Under **Settings** → **Subnets** → **Associate**, I linked it to:

| Setting | Value |
|---|---|
| Virtual network | CoreServicesVnet (az104-rg4) |
| Subnet | SharedServicesSubnet |

<img width="1297" height="172" alt="image" src="https://github.com/user-attachments/assets/ada6a3c8-a027-4c24-8bba-09070fc95ece" />

**Inbound rule — allow ASG traffic:**

Under **Inbound security rules**, the default rules only allow traffic from other virtual networks and load balancers — nothing from the ASG yet. Added a new rule:

| Setting | Value |
|---|---|
| Source | Application security group |
| Source application security groups | asg-web |
| Source port ranges | * |
| Destination | Any |
| Service | Custom |
| Destination port ranges | 80,443 |
| Protocol | TCP |
| Action | Allow |
| Priority | 100 |
| Name | AllowASG |

<img width="504" height="761" alt="image" src="https://github.com/user-attachments/assets/92d2e4aa-c1b9-47f5-bcad-ba1b332d6ba0" />

**Outbound rule — deny internet access:**

The default **AllowInternetOutBound** rule sits at priority 65001 and can't be deleted, so instead I added a higher-priority rule to override it for outbound internet traffic specifically:

| Setting | Value |
|---|---|
| Source | Any |
| Source port ranges | * |
| Destination | Service tag |
| Destination service tag | Internet |
| Service | Custom |
| Destination port ranges | * |
| Protocol | Any |
| Action | Deny |
| Priority | 4096 |
| Name | DenyInternetOutbound |

<img width="521" height="774" alt="image" src="https://github.com/user-attachments/assets/cde811cb-1ced-4989-9526-e3fad0881fbf" />

---

## Task 4: Configure Public and Private Azure DNS Zones

Last task covered both kinds of DNS zones Azure supports — public, for resolving names on the open internet, and private, for resolution that only works inside linked virtual networks.

**Public DNS zone:**

Searched for **DNS zones**, hit **+ Create**:

| Setting | Value |
|---|---|
| Subscription | my subscription |
| Resource group | az104-rg4 |
| Name | salnoct.com |
| Region | East US |

**Review + create** → **Create** → **Go to resource**.

On the Overview blade, Azure assigns four DNS name servers to the zone — I copied one of those addresses for testing later.

<img width="447" height="126" alt="image" src="https://github.com/user-attachments/assets/dea1e815-052f-4b73-87e6-9ef5b8f9ec4a" />

Under **DNS Management** → **Recordsets**, I added an A record:

| Setting | Value |
|---|---|
| Name | www |
| Type | A |
| Alias record set | No |
| TTL | 1 |
| IP address | 10.1.1.4 |

> In an actual production setup, that IP would be the real public IP of a web server — here it's just a placeholder to confirm resolution works.

To verify, I ran `nslookup` against the name server I'd copied earlier and confirmed the hostname resolved to the IP address I'd set.

<img width="451" height="110" alt="image" src="https://github.com/user-attachments/assets/376ea606-6b06-4dc6-bf87-48cb88147f24" />

**Private DNS zone:**

A private DNS zone only resolves names inside the virtual networks it's linked to — not reachable from the public internet at all.

Searched for **Private DNS zones**, hit **+ Create**:

| Setting | Value |
|---|---|
| Subscription | my subscription |
| Resource group | az104-rg4 |
| Name | private.salnoct.com |
| Region | East US |

**Review + create** → **Create** → **Go to resource**. Unlike the public zone, there are no name server records shown here — that's expected for a private zone.

<img width="687" height="556" alt="image" src="https://github.com/user-attachments/assets/1e33c616-2aae-40e9-b430-0a4496fe0a79" />

Under **DNS Management** → **Virtual network links**, I linked it to the Manufacturing network:

| Setting | Value |
|---|---|
| Link name | manufacturing-link |
| Virtual network | ManufacturingVnet |

**Create**, then waited for the link to finish provisioning.

From **DNS Management** → **+ Recordsets**, added a record for a VM that would need private name resolution:

| Setting | Value |
|---|---|
| Name | sensorvm |
| Type | A |
| TTL | 1 |
| IP address | 10.1.1.4 |

> Again, a placeholder IP here — in a real environment this would point at an actual manufacturing VM, and you'd repeat this for every machine that needs private resolution.

---

## Cleanup

If running this on a real subscription, delete the resource group afterward to avoid leftover cost:

- Azure portal: select the resource group → **Delete the resource group** → enter the name to confirm → **Delete**
- PowerShell: `Remove-AzResourceGroup -Name resourceGroupName`
- CLI: `az group delete --name resourceGroupName`

---

## Key Takeaways

- A **virtual network** is basically your own private network, recreated in the cloud.
- Avoiding **overlapping IP address ranges** between networks keeps things simpler to manage and troubleshoot later.
- A **subnet** is a slice of IP addresses within a virtual network, used to split things up for organization and security.
- A **network security group (NSG)** holds rules that allow or deny traffic — there are sensible defaults, but you can customize them.
- An **application security group (ASG)** groups servers that share a function (web servers, database servers, etc.) so NSG rules can target the group instead of individual IPs.
- **Azure DNS** handles name resolution for both public domains and private, VNet-scoped name resolution.
