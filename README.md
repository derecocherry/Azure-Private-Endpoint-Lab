# Azure Private Endpoint Lab

## Overview

This hands-on lab demonstrates how to configure an **Azure Private Endpoint** for an Azure Storage Account and access a private blob from a virtual machine.

The goal was to understand how these Azure services work together:

* Azure Virtual Network (VNet)
* Subnets
* Azure Virtual Machine
* Azure Storage Account
* Private Endpoint
* Azure Private DNS
* Microsoft Entra ID authentication
* Azure RBAC

The final test proved that the VM could access a blob while **Public Network Access was disabled** on the Storage Account.

---

## Lab Architecture

```text
                         Azure
                           │
                    ┌──────▼──────┐
                    │   Storage   │
                    │   Account   │
                    └──────▲──────┘
                           │
                      Private Link
                           │
                    ┌──────┴──────┐
                    │   Private   │
                    │   Endpoint  │
                    │  10.0.2.4   │
                    └──────▲──────┘
                           │
                     Private DNS
                           │
             ┌─────────────┴─────────────┐
             │           VNet            │
             │       10.0.0.0/16         │
             │                           │
             │  ┌─────────────────────┐  │
             │  │      VM Subnet      │  │
             │  │     10.0.1.0/24     │  │
             │  │                     │  │
             │  │     Windows VM      │  │
             │  │       10.0.1.x      │  │
             │  └─────────────────────┘  │
             │                           │
             │  ┌─────────────────────┐  │
             │  │ Private Endpoint    │  │
             │  │      Subnet         │  │
             │  │     10.0.2.0/24     │  │
             │  └─────────────────────┘  │
             └───────────────────────────┘
```

---

# Lab Environment

## Resource Group

```text
rg-privateendpoint-lab
```

Region:

```text
East US
```

---

## Virtual Network

Name:

```text
vnet-pe-lab
```

Address Space:

```text
10.0.0.0/16
```

### Subnets

| Subnet                  | Address Range | Purpose          |
| ----------------------- | ------------- | ---------------- |
| `snet-vm`               | `10.0.1.0/24` | Windows VM       |
| `snet-private-endpoint` | `10.0.2.0/24` | Private Endpoint |

---

# Step 1 — Create the Resource Group

Create a resource group named:

```text
rg-privateendpoint-lab
```

Use:

```text
Region: East US
```

---

# Step 2 — Create the Virtual Network

Create:

```text
vnet-pe-lab
```

Address space:

```text
10.0.0.0/16
```

Create two subnets.

### VM Subnet

```text
Name: snet-vm
Address range: 10.0.1.0/24
```

### Private Endpoint Subnet

```text
Name: snet-private-endpoint
Address range: 10.0.2.0/24
```

Keeping the VM and Private Endpoint in separate subnets makes the network architecture easier to understand and manage.

---

# Step 3 — Create the Windows Virtual Machine

Create a Windows Server 2022 VM.

Example:

```text
VM Name: vm-pe-test
```

Place the VM in:

```text
VNet: vnet-pe-lab
Subnet: snet-vm
```

The VM received a private IP address in the:

```text
10.0.1.0/24
```

network.

A temporary public IP was used to connect to the VM using RDP.

---

# Step 4 — Verify VM Internet Connectivity

Connect to the VM using RDP.

Run:

```powershell
Test-NetConnection www.microsoft.com -Port 443
```

Expected result:

```text
TcpTestSucceeded : True
```

This confirms the VM has outbound network connectivity.

---

# Step 5 — Create the Storage Account

Create an Azure Storage Account in:

```text
Resource Group: rg-privateendpoint-lab
Region: East US
Performance: Standard
Redundancy: LRS
```

Keep the exact Storage Account name unique to the environment.

---

# Step 6 — Create a Blob Container

Inside the Storage Account, create:

```text
Container: test
```

Set the container to:

```text
Private
```

Do not allow anonymous public access.

Upload a test file:

```text
private-endpoint-test.txt
```

---

# Step 7 — Create the Private Endpoint

Create a Private Endpoint:

```text
Name: pe-storage-lab
```

Configure it for the Storage Account.

Target sub-resource:

```text
blob
```

Network:

```text
VNet: vnet-pe-lab
Subnet: snet-private-endpoint
```

Enable:

```text
Private DNS integration
```

Azure creates/configures the Private DNS zone:

```text
privatelink.blob.core.windows.net
```

The Private Endpoint received the private IP:

```text
10.0.2.4
```

---

# Step 8 — Test Private DNS

From the Windows VM, run:

```powershell
nslookup YOURSTORAGEACCOUNT.blob.core.windows.net
```

The Storage Account hostname should resolve to the Private Endpoint IP.

Expected:

```text
Address: 10.0.2.4
```

This is an important test.

The VM is using the normal Storage Account hostname, but DNS is directing the connection to the Private Endpoint.

---

# Step 9 — Test HTTPS Connectivity

From the VM:

```powershell
Test-NetConnection YOURSTORAGEACCOUNT.blob.core.windows.net -Port 443
```

Expected:

```text
RemoteAddress    : 10.0.2.4
TcpTestSucceeded : True
```

This proves that the VM can reach the Storage Account through the Private Endpoint.

---

# Step 10 — Disable Public Network Access

In the Storage Account:

```text
Networking
```

Change:

```text
Public network access
```

to:

```text
Disabled
```

Save the configuration.

This creates an important test.

If the Private Endpoint is configured correctly, the VM should still be able to reach the Storage Account.

---

# Step 11 — Test Connectivity Again

From the VM:

```powershell
Test-NetConnection YOURSTORAGEACCOUNT.blob.core.windows.net -Port 443
```

Expected:

```text
RemoteAddress    : 10.0.2.4
TcpTestSucceeded : True
```

The connection continued to work even though public network access was disabled.

This demonstrates the private network path.

---

# Step 12 — Understand the Browser Test

Attempting to browse directly to:

```text
https://YOURSTORAGEACCOUNT.blob.core.windows.net/test
```

returned a message indicating that public access was not permitted.

This was expected.

The container was private and public network access was disabled.

Network connectivity and data authorization are two different things.

---

# Step 13 — Install Azure Storage Explorer

Install Microsoft Azure Storage Explorer on the VM.

Sign in using the Microsoft Entra account associated with the Azure subscription.

---

# Step 14 — Troubleshooting Storage Explorer

Initially, the `test` container did not appear in Storage Explorer.

The Azure Portal showed that the container existed.

Storage Explorer displayed a message indicating that the tenant was filtered out.

### Resolution

Open the Storage Explorer tenant configuration and remove the filter for the tenant associated with the Azure subscription.

After refreshing Storage Explorer, the following appeared:

```text
Blob Containers
└── test
```

---

# Step 15 — Configure Blob Data Permissions

The Azure account initially had:

```text
Owner
```

on the Storage Account.

However, Azure management-plane permissions and Storage data-plane permissions are different.

Add:

```text
Storage Blob Data Contributor
```

to the account used by Storage Explorer.

This provides permissions to work with the actual blob data.

---

# Step 16 — Access the Blob

In Storage Explorer:

```text
Storage Accounts
└── Lab Storage Account
    └── Blob Containers
        └── test
            └── private-endpoint-test.txt
```

The blob was successfully opened from the VM.

---

# Final Validation

The lab successfully demonstrated:

### DNS

```text
YOURSTORAGEACCOUNT.blob.core.windows.net
                 ↓
              10.0.2.4
```

### Network

```text
VM
 ↓
Private DNS
 ↓
10.0.2.4
 ↓
Private Endpoint
 ↓
Azure Storage
```

### Authentication

Microsoft Entra authentication was used through Storage Explorer.

### Authorization

The user was assigned:

```text
Storage Blob Data Contributor
```

### Public Access

Storage Account:

```text
Public Network Access: Disabled
```

### Final Result

The VM was still able to access:

```text
private-endpoint-test.txt
```

---

# Key Azure Concepts Learned

## Private Endpoint

A Private Endpoint creates a network interface with a **private IP address** inside an Azure VNet.

In this lab:

```text
Private Endpoint IP: 10.0.2.4
```

The Azure Storage service can therefore be accessed through a private network path.

---

## Private DNS

Private DNS allows the normal Azure Storage hostname to resolve to the Private Endpoint's private IP.

Without the appropriate DNS configuration, clients may resolve the Storage Account to a public endpoint instead.

In this lab:

```text
Storage hostname
       ↓
Private DNS
       ↓
10.0.2.4
```

---

## Azure RBAC

Azure RBAC controls what a user can do.

There is an important distinction between:

### Management Plane

Example:

```text
Owner
```

This provides management permissions over the Azure resource.

### Data Plane

Example:

```text
Storage Blob Data Contributor
```

This provides permissions to work with the actual blob data.

---

# Private Endpoint vs. Service Endpoint

| Feature                                              | Service Endpoint                 | Private Endpoint                   |
| ---------------------------------------------------- | -------------------------------- | ---------------------------------- |
| Private IP in VNet                                   | No                               | Yes                                |
| Uses Private Link                                    | No                               | Yes                                |
| Private DNS commonly required                        | No                               | Yes                                |
| Resource gets network interface                      | No                               | Yes                                |
| Can disable public access and still use private path | Depends on service/configuration | Yes                                |
| Resource appears directly in VNet address space      | No                               | Private IP represents the endpoint |

A simple way to remember it:

> **Service Endpoint = private access path without a private IP for the service**

> **Private Endpoint = private IP for the service inside your VNet**

---

# Troubleshooting Lessons

## Problem: `test` container missing in Storage Explorer

### Cause

The Azure tenant was filtered out.

### Solution

Open the Storage Explorer tenant configuration and unfilter the correct tenant.

---

## Problem: Storage Account accessible but blobs cannot be accessed

### Possible cause

The account has Azure resource permissions but lacks Storage data-plane permissions.

### Solution

Assign an appropriate Storage Blob Data role, such as:

```text
Storage Blob Data Contributor
```

---

# AZ-104 Mental Model

When a VM accesses an Azure Storage Account through a Private Endpoint, think about the process in this order:

```text
1. VM needs Storage
        ↓
2. VM uses Storage hostname
        ↓
3. DNS resolves the hostname
        ↓
4. Private DNS returns Private Endpoint IP
        ↓
5. VM connects to the private IP
        ↓
6. Private Endpoint uses Private Link
        ↓
7. Storage receives the traffic
        ↓
8. Azure RBAC determines data access
```

The most important concepts to remember are:

```text
Private Endpoint = Network
Private DNS      = Name Resolution
RBAC             = Authorization
```

---

# Cleanup

When finished with the lab, delete the entire resource group:

```text
rg-privateendpoint-lab
```

This removes the resources created for the lab and helps avoid unnecessary Azure charges.

---

# What I Learned

This lab helped demonstrate that Azure Private Endpoint is not just about creating an endpoint.

A complete implementation involves:

* Network design
* VNet/subnet configuration
* Private Endpoint
* Private DNS
* DNS resolution
* Network connectivity
* Storage networking configuration
* Microsoft Entra authentication
* Azure RBAC
* Troubleshooting

The most important validation was successfully accessing the blob after **Public Network Access was disabled**.

---

## Lab Status

**Completed:** ✅

**Private Endpoint:** ✅

**Private DNS:** ✅

**Private Connectivity:** ✅

**Public Network Access Disabled:** ✅

**Microsoft Entra Authentication:** ✅

**Blob RBAC:** ✅

**Blob Access:** ✅
