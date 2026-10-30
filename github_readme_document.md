# AWS Multi-VPC Architecture & Peering Connection

A comprehensive guide and demonstration for setting up a custom multi-VPC architecture on Amazon Web Services (AWS) featuring Public and Private Subnets, Internet Gateways, Route Tables, and a cross-VPC Peering Connection.

---

## 📌 Project Overview

This repository documents the implementation of a multi-VPC network infrastructure built for the **EWB Tech Academy AWS Internship Program**. 

The goal of this project is to demonstrate secure, isolated private communication between instances located in two separate Virtual Private Clouds (**VPC 1** and **VPC 2**) using **AWS VPC Peering**, while maintaining controlled public internet entry via a **Bastion / Jump Host** in a public subnet.

---

## 🏗️ Architecture & Network Design

```
                     +------------------------------------------------------------------+
                     |                          AWS CLOUD                               |
                     |                                                                  |
                     |   +----------------------------------------------------------+   |
                     |   | VPC 1 (Primary): 28.0.0.0/16                             |   |
                     |   |                                                          |   |
                     |   |  +------------------------+    +----------------------+  |   |
  [ INTERNET ] <=======> |  | Public Subnet          |    | Private Subnet       |  |   |
       ||            |   |  | 28.0.1.0/24            |    | 28.0.2.0/24          |  |   |
       ||            |   |  |                        |    |                      |  |   |
       ||            |   |  |  +------------------+  |    |  +----------------+  |  |   |
       ||            |   |  |  | Public EC2       |  |    |  | Private EC2    |  |  |   |
       ||            |   |  |  | (Bastion Host)   |  |    |  | (App Server)   |  |  |   |
       ||            |   |  |  +------------------+  |    |  +----------------+  |  |   |
       ||            |   +--+------------------------+----+----------------------+--+   |
       ||            |      | Internet Gateway (IGW) |                                  |
       ||            |      +------------------------+                                  |
       ||            |                                 ||                               |
       ||            +---------------------------------||--------------------------------+
       ||                                              || (VPC Peering Connection: pcx-xxxxxx)
       ||            +---------------------------------||--------------------------------+
       ||            | VPC 2 (Target): 28.200.0.0/16   ||                               |
       ||            |                                 \/                               |
       ||            |   +----------------------------------------------------------+   |
       ||            |   | Private Subnet                                           |   |
       ||            |   | 28.200.0.0/24                                            |   |
       ||            |   |                                                          |   |
       ||            |   |  +----------------------------------------------------+  |   |
       ||            |   |  | Private EC2 Instance (Target Server)               |  |   |
       ||            |   |  +----------------------------------------------------+  |   |
       ||            |   +----------------------------------------------------------+   |
       ||            +------------------------------------------------------------------+
```

### Network Topology Breakdown

| Resource Name | VPC Assignment | IPv4 CIDR Block | Subnet Type | Target Routes |
| :--- | :--- | :--- | :--- | :--- |
| **VPC-1-Primary** | VPC 1 | `28.0.0.0/16` | N/A | Local |
| **Public-Subnet-1** | VPC 1 | `28.0.1.0/24` | Public | `0.0.0.0/0` $\rightarrow$ IGW, `28.200.0.0/16` $\rightarrow$ PCX |
| **Private-Subnet-1** | VPC 1 | `28.0.2.0/24` | Private | Local, `28.200.0.0/16` $\rightarrow$ PCX |
| **VPC-2-Target** | VPC 2 | `28.200.0.0/16` | N/A | Local |
| **Target-Private-Subnet**| VPC 2 | `28.200.0.0/24` | Private | Local, `28.0.0.0/16` $\rightarrow$ PCX |
| **IGW-VPC-1** | VPC 1 | Attached to VPC 1 | Gateway | Attached to Public Subnet |
| **PCX-VPC1-VPC2** | Both VPCs | Peers 28.0.0.0/16 & 28.200.0.0/16 | Peering Link | Status: `Active` |

---

## 🚀 Step-by-Step Implementation Guide

### Step 1: Create VPCs & Subnets
1. **Create VPC 1 (Primary):**
   * Go to **VPC Dashboard** $\rightarrow$ **Create VPC**.
   * Select **VPC only**.
   * **Name tag:** `VPC-1-Primary` | **IPv4 CIDR block:** `28.0.0.0/16`.
2. **Create VPC 2 (Target):**
   * Click **Create VPC** $\rightarrow$ **VPC only**.
   * **Name tag:** `VPC-2-Target` | **IPv4 CIDR block:** `28.200.0.0/16`.
3. **Create Subnets:**
   * **Public Subnet (VPC 1):** Select `VPC-1-Primary` $\rightarrow$ Name: `Public-Subnet-1` $\rightarrow$ CIDR: `28.0.1.0/24`.
   * **Private Subnet (VPC 1):** Select `VPC-1-Primary` $\rightarrow$ Name: `Private-Subnet-1` $\rightarrow$ CIDR: `28.0.2.0/24`.
   * **Private Subnet (VPC 2):** Select `VPC-2-Target` $\rightarrow$ Name: `Target-Private-Subnet` $\rightarrow$ CIDR: `28.200.0.0/24`.
4. **Enable Auto-Assign Public IP:**
   * Select `Public-Subnet-1` $\rightarrow$ **Actions** $\rightarrow$ **Edit subnet settings**.
   * Check **Enable auto-assign public IPv4 address** and save.

---

### Step 2: Configure Internet Gateway & Route Tables
1. **Create & Attach Internet Gateway:**
   * Go to **Internet Gateways** $\rightarrow$ **Create internet gateway** $\rightarrow$ Name: `IGW-VPC-1`.
   * Click **Actions** $\rightarrow$ **Attach to VPC** $\rightarrow$ Select `VPC-1-Primary`.
2. **Configure Public Route Table (VPC 1):**
   * Go to **Route Tables** $\rightarrow$ **Create route table** $\rightarrow$ Name: `Public-RT-VPC1` $\rightarrow$ Select `VPC-1-Primary`.
   * Edit Routes: Add Destination `0.0.0.0/0` targeting `IGW-VPC-1`.
   * Subnet Associations: Associate with `Public-Subnet-1`.
3. **Configure Private Route Tables:**
   * Create `Private-RT-VPC1` for `VPC-1-Primary` and associate with `Private-Subnet-1`.
   * Create `Private-RT-VPC2` for `VPC-2-Target` and associate with `Target-Private-Subnet`.

---

### Step 3: Establish VPC Peering Connection
1. **Initiate Request (Requester - VPC 1):**
   * Go to **Peering Connections** $\rightarrow$ **Create Peering Connection**.
   * Name: `PCX-VPC1-VPC2`.
   * Requester: Select `VPC-1-Primary`.
   * Accepter: Select `VPC-2-Target` (Same account / region or cross-account/region).
2. **Accept Request (Accepter - VPC 2):**
   * Select `PCX-VPC1-VPC2` (Status: `Pending Acceptance`).
   * Click **Actions** $\rightarrow$ **Accept Request**. Status becomes **Active**.
3. **Update Route Tables for Peering:**
   * **In `Public-RT-VPC1` & `Private-RT-VPC1`:** Add route `28.200.0.0/16` targeting `PCX-VPC1-VPC2`.
   * **In `Private-RT-VPC2`:** Add route `28.0.0.0/16` targeting `PCX-VPC1-VPC2`.

---

### Step 4: Deploy EC2 Instances & Security Groups
1. **Public Bastion Host (VPC 1):**
   * Launch Amazon Linux 2 EC2 instance in `Public-Subnet-1`.
   * **Security Group Rules:** Allow SSH (Port 22) from `My IP` and HTTP (Port 80) from `0.0.0.0/0`.
2. **Target Private EC2 (VPC 2):**
   * Launch Amazon Linux 2 EC2 instance in `Target-Private-Subnet`.
   * **Security Group Rules:** Allow **All ICMP - IPv4** from `28.0.0.0/16` and **SSH (Port 22)** from `28.0.0.0/16`.

---

## 🧪 Connectivity Testing & Verification

1. **Connect to Public Bastion Host via SSH:**
   ```bash
   ssh -i keypair.pem ec2-user@<PUBLIC_IP_ADDRESS>
   ```

2. **Upload SSH Keypair & Set Permissions:**
   ```bash
   chmod 600 keypair.pem
   ```

3. **SSH into Private EC2 in VPC 2 across Peering Link:**
   ```bash
   ssh -i keypair.pem ec2-user@28.200.0.X
   ```

4. **Verify ICMP Ping Across Peered Network:**
   ```bash
   ping 28.200.0.X
   ```
   *Successful ICMP responses confirm operational routing tables and active VPC Peering!*

---

## 📜 Submission Checklist

- [x] **VPC 1 & VPC 2 Created** with non-overlapping CIDRs.
- [x] **Public & Private Subnets Created** with proper route table associations.
- [x] **Internet Gateway Attached** to VPC 1 public subnet.
- [x] **VPC Peering Connection Created & Accepted**.
- [x] **Cross-VPC Routes Added** pointing to Peering ID (`pcx-xxxxxx`).
- [x] **Security Groups Configured** for SSH/ICMP access.
- [x] **Jump Host SSH & Ping Verified**.

---
*Created as part of the final project submission for EWB Tech Academy Internship.*