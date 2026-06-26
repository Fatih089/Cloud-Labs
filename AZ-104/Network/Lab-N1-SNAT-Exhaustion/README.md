# Lab N1 - SNAT Exhaustion: NAT Gateway vs. Standard Load Balancer

## Overview

This lab demonstrates how Azure NAT Gateway improves outbound connectivity compared to a Standard Load Balancer. It also explains how SNAT port exhaustion can occur under high outbound traffic and why Azure recommends using a NAT Gateway for scalable outbound connections.

## Lab Objectives

After completing this lab, you will be able to:

- Understand how SNAT ports are used for outbound connectivity
- Explain the limitations of a Standard Load Balancer
- Deploy and configure an Azure NAT Gateway
- Associate a NAT Gateway with a subnet
- Validate outbound connectivity before and after deploying a NAT Gateway

## Azure Services

- Azure Resource Group
- Azure Virtual Network
- Azure Subnet
- Azure Virtual Machine Scale Set
- Azure Standard Load Balancer
- Azure NAT Gateway
- Azure Public IP Address

## Step 1 - Prepare the Environment

### Create a Resource Group

| Setting | Value |
|---------|-------|
| Name | `rg-n1-snat` |
| Region | `West Europe` |

### Create a Virtual Network

| Setting | Value |
|---------|-------|
| Name | `vnet-n1` |
| Address Space | `10.10.0.0/16` |

Create the following subnet:

| Setting | Value |
|---------|-------|
| Name | `vm-subnet` |
| Address Range | `10.10.1.0/24` |

## Step 2 - Deploy a Virtual Machine Scale Set

Deploy a Virtual Machine Scale Set without a NAT Gateway.

### Basic Settings

| Setting | Value |
|---------|-------|
| Name | `vmss-n1` |
| Image | Ubuntu LTS |
| Instances | 2-3 |
| Authentication | SSH Key or Password |

### Networking

| Setting | Value |
|---------|-------|
| Virtual Network | `vnet-n1` |
| Subnet | `vm-subnet` |
| Public IP | None |
| Load Balancer | Standard Load Balancer |

At this stage, Azure provides outbound connectivity through the Standard Load Balancer using SNAT.

## Step 3 - Generate Outbound Traffic

Connect to one of the VMSS instances using Azure Bastion or a jump box.

Generate a large number of outbound connections:

```bash
for i in {1..20000}; do curl -m 1 https://microsoft.com >/dev/null 2>&1 & done
```

Alternative:

```bash
seq 1 20000 | xargs -n1 -P200 curl -m 1 https://microsoft.com >/dev/null 2>&1
```

## Step 4 - Observe SNAT Behavior

A Standard Load Balancer provides only a limited number of outbound SNAT ports for each backend instance.

Under heavy outbound traffic you may observe:

- Connection timeouts
- Failed outbound connections
- SNAT port exhaustion

Check the current outbound public IP:

```bash
curl ifconfig.me
```

Expected result:

The returned IP address should be the public frontend IP of the Standard Load Balancer.

## Step 5 - Deploy an Azure NAT Gateway

### Create a Public IP Address

| Setting | Value |
|---------|-------|
| Name | `pip-nat` |
| SKU | Standard |
| Assignment | Static |

### Create the NAT Gateway

| Setting | Value |
|---------|-------|
| Name | `nat-n1` |
| Public IP | `pip-nat` |
| Idle Timeout | 4 Minutes |

### Associate the NAT Gateway

Associate the NAT Gateway with the `vm-subnet` subnet.

> Note: Azure NAT Gateway is associated with a subnet, not with individual virtual machines.

## Step 6 - Validate the Solution

Run the outbound traffic test again:

```bash
for i in {1..20000}; do curl -m 1 https://microsoft.com >/dev/null 2>&1 & done
```

Check the outbound IP address again:

```bash
curl ifconfig.me
```

Expected results:

- The outbound IP address should now be the NAT Gateway Public IP
- Outbound connectivity should remain stable
- Significantly more SNAT ports are available

## Architecture

```text
Internet
    |
    v
Public IP
    |
    v
NAT Gateway
    |
    v
Virtual Network
    |
    v
Subnet
    |
    v
Virtual Machine Scale Set
```

## Key Takeaways

### Standard Load Balancer

- Provides limited outbound SNAT ports
- Suitable for general workloads
- May experience SNAT exhaustion under high outbound traffic

### Azure NAT Gateway

- Designed for scalable outbound connectivity
- Provides up to 64,000 SNAT ports per Public IP
- Recommended for workloads with high outbound connection requirements

## Security Considerations

- No inbound Public IP is assigned to the VM Scale Set
- Outbound connectivity is managed using Azure NAT Gateway
- Standard SKU networking resources are used

## Cost Considerations

To minimize Azure costs:

- Delete the Resource Group after completing the lab
- Use only the required number of VM instances
- Stop or remove resources when they are no longer needed

## Cleanup

Delete all resources after completing the lab:

```bash
az group delete --name rg-n1-snat --yes --no-wait
```
