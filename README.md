# Azure Network Isolation and Secure Storage Lab

## Overview

A company needs its database tier fully isolated from the public internet, a web tier that only accepts HTTPS traffic, encrypted storage for sensitive data, and a way to redeploy that same environment consistently instead of rebuilding it by hand each time. This project builds that environment in Azure: a segmented virtual network, a locked-down storage account reachable only through a private connection, and both resources captured as reusable deployment templates.

**Scope:** 1 resource group, 1 virtual network with 2 subnets, 1 network security group with 3 rules, 1 storage account secured with a private endpoint, 2 resources captured and redeployed as ARM templates, and network isolation verified end to end with a live access test.

## Skills & Tools

| Category | Tools/Skills |
|---|---|
| Cloud Infrastructure | Microsoft Azure, Resource Groups, Virtual Networks, Subnetting |
| Network Security | Network Security Groups, rule priority and precedence, Private Endpoints, Service Endpoints |
| Storage | Azure Blob Storage, encryption, redundancy (LRS), Shared Access Signatures |
| Infrastructure as Code | ARM Templates, Template Specs, repeatable deployment |
| Verification | Azure Cloud Shell, AzCopy, network isolation testing |

## Environment

- **Subscription:** Azure for Students
- **Resource Group:** `RG-NetStorage-Lab`
- **Virtual Network:** `Vnet` (10.0.0.0/16)
  - **Database subnet:** 10.0.0.0/27
  - **Web subnet:** 10.0.1.0/27
- **Network Security Group:** `web-NSG`
- **Storage Account:** `db4website` (Blob storage, LRS, public access disabled)
- **Private Endpoint:** `database_EP`, scoped to the Database subnet
- **Region:** East US 2

## Architecture

The Web and Database subnets sit in the same VNet but serve different purposes: Web is the only subnet meant to accept inbound traffic from the internet, and only on port 443. Database is where the storage account's Private Endpoint lives, so anything reaching the storage account has to originate from inside that subnet, not from the public internet. This mirrors a real segmented design where a public-facing tier and a data tier are kept on separate trust boundaries, even inside the same network.

## Steps

### 1. Created a Virtual Network with two subnets

Set up `Vnet` with a Database subnet (10.0.0.0/27) and a Web subnet (10.0.1.0/27), each sized for exactly what this lab needed rather than defaulting to a larger block.

<img src="screenshots/01_vnet-subnets-created.png" width="700"><br><br>

### 2. Created a Network Security Group

Deployed `web-NSG` to control inbound traffic to the Web subnet.

<img src="screenshots/02_nsg-created.png" width="700"><br><br>

### 3. Associated the NSG with the Web subnet

<img src="screenshots/03_nsg-associated-with-web-subnet.png" width="700"><br><br>

### 4. Configured inbound security rules

Set up three rules: allow HTTPS (443) at priority 100, explicitly deny RDP (3389) at priority 110, and deny all other inbound traffic at priority 200. RDP was called out and denied explicitly rather than left to fall through to the catch-all rule, to make the intent visible rather than incidental.

<img src="screenshots/04_nsg-inbound-rules-corrected.png" width="900"><br><br>

### 5. Created a Storage Account

Deployed `db4website` as a general-purpose v2 account with Blob storage as the primary service and locally redundant storage (LRS), since this is a lab environment rather than production data requiring geo-redundancy.

<img src="screenshots/05_storage-account-basics.png" width="700"><br><br>

### 6. Disabled public network access on the Storage Account

<img src="screenshots/06_storage-account-networking-disabled.png" width="900"><br><br>

### 7. Created a Private Endpoint scoped to the Database subnet

Gave the storage account a private IP address inside the Database subnet, so it can only be reached from within that subnet rather than over the public internet.

<img src="screenshots/07_private-endpoint-database-subnet.png" width="900"><br><br>

### 8. Exported the Virtual Network as an ARM template

<img src="screenshots/08_vnet-export-template.png" width="900"><br><br>

### 9. Imported both templates into Template Specs

Captured the VNet and Storage Account configurations as versioned Template Specs, so either resource can be redeployed consistently without manually reconfiguring it through the portal.

<img src="screenshots/09_template-specs-list.png" width="700"><br><br>

### 10. Deployed the VNet from its Template Spec

<img src="screenshots/10_vnet-template-deploy.png" width="700"><br><br>

### 11. Confirmed the VNet deployment completed successfully

<img src="screenshots/11_vnet-deployment-complete.png" width="900"><br><br>

### 12. Deployed the Storage Account from its Template Spec

<img src="screenshots/12_storage-template-deploy.png" width="700"><br><br>

### 13. Confirmed the Storage Account deployment completed successfully

<img src="screenshots/13_storage-deployment-complete.png" width="700"><br><br>

### 14. Generated a Shared Access Signature to test access

<img src="screenshots/14_sas-token-generation.png" width="700"><br><br>

### 15. Attempted an AzCopy upload from outside the Virtual Network

Ran an AzCopy upload from Azure Cloud Shell, which sits outside the VNet, using a valid SAS token. The request was denied with a 403 Authorization Failure, confirming that the Private Endpoint correctly blocks access from anywhere other than the Database subnet, regardless of whether the credentials presented are valid.

<img src="screenshots/15_azcopy-test-403-private-endpoint-blocking.png" width="900"><br><br>

## Decisions & Significance

- **Right-sized subnets instead of default-sized ones.** Each subnet was sized at /27 (32 addresses) rather than a much larger default block, matching the actual number of resources expected in this environment instead of over-allocating address space.

- **A Private Endpoint over a Service Endpoint.** A Service Endpoint keeps traffic on Azure's private backbone, but the storage account still has a public IP behind it. A Private Endpoint goes further, giving the storage account an actual private IP inside the VNet, which is what fully removes it from the public internet. This project uses a Private Endpoint specifically because the requirement was true isolation, not just a faster private route.

- **RDP denied explicitly, not just caught by the catch-all rule.** The deny-all rule at priority 200 would have blocked RDP anyway, but adding a named rule that explicitly denies port 3389 makes the security intent visible in the rule table itself, rather than leaving a reader to infer it from a generic catch-all.

- **LRS over GRS for this environment.** Since this is a lab rather than a production deployment protecting live customer data, LRS was chosen to minimize cost. A production version of this same architecture protecting real business data would more likely justify GRS or RA-GRS for regional redundancy.

- **ARM templates captured via Template Specs rather than raw JSON files.** Exporting a template proves the configuration can be captured as code, but storing it as a Template Spec turns it into a reusable, versioned Azure resource that can be redeployed on demand, closer to how a team would actually manage infrastructure as code rather than passing JSON files around manually.

## Troubleshooting Notes

- NSG rules were initially misconfigured with HTTPS denied and RDP allowed, the reverse of the intended policy; caught during review and corrected to allow HTTPS and explicitly deny RDP
- Redeploying the storage account from its Template Spec initially failed because the storage account name was already taken, since Azure requires globally unique storage account names; resolved by targeting the same resource group as the original account
- AzCopy upload from Cloud Shell failed with a 403 Authorization Failure despite a valid, correctly scoped SAS token; this was expected behavior, not a bug, since the storage account's public network access is disabled and Cloud Shell runs outside the VNet, confirming the Private Endpoint's network-level restriction takes precedence over token-level permissions

## What This Demonstrates

- Designing network segmentation around trust boundaries, not just convenience
- Enforcing least-privilege network access with NSG rules, including explicit denies for sensitive ports
- Using Private Endpoints to remove a PaaS resource from the public internet entirely
- Capturing infrastructure as reusable, versioned deployment templates
- Validating a security control by testing that it actually fails closed, not just assuming it works

## Why This Project Matters

A misconfigured network boundary or an exposed storage account is one of the most common ways company data ends up reachable from outside its intended perimeter. This project shows a practical way to prevent that: segmenting traffic by tier, removing a sensitive resource from the public internet entirely, and proving that isolation holds even when a valid credential is presented from the wrong location.
