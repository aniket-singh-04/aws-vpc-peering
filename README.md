# AWS VPC Peering Project

A hands-on AWS networking project demonstrating how to establish private communication between two Virtual Private Clouds (VPCs) using **VPC Peering, custom route tables, security groups, and EC2 instances**.

The project follows a practical AWS networking lab setup in the Mumbai region (`ap-south-1`).

## Table of Contents

- [Project Overview](#project-overview)
- [Objectives](#objectives)
- [Architecture](#architecture)
- [AWS Resources](#aws-resources)
- [IP Addressing Plan](#ip-addressing-plan)
- [Network Configuration](#network-configuration)
- [Security Configuration](#security-configuration)
- [Implementation Steps](#implementation-steps)
- [Testing and Validation](#testing-and-validation)
- [Repository Structure](#repository-structure)
- [Security Best Practices](#security-best-practices)
- [Cleanup](#cleanup)
- [Learning Outcomes](#learning-outcomes)

## Project Overview

Amazon VPC enables you to create logically isolated virtual networks in AWS. VPC Peering allows resources in two VPCs to communicate using private IP addresses without requiring traffic to traverse the public internet.

In this project, two VPCs are connected through a VPC Peering connection.

- **VPC A:** Contains a public subnet, a private subnet, a bastion host, and an application EC2 instance.
- **VPC B:** Contains a private subnet and another application EC2 instance.
- **VPC Peering:** Enables private communication between the application instances in the two VPCs.
- **Bastion Host:** Provides an SSH entry point for securely accessing private EC2 instances.

## Objectives

- Create two custom VPCs with non-overlapping CIDR blocks.
- Configure public and private subnets.
- Create and attach an Internet Gateway to VPC A.
- Configure custom route tables.
- Launch EC2 instances in public and private subnets.
- Establish a VPC Peering connection between VPC A and VPC B.
- Configure routes for communication between the VPCs.
- Apply security group rules to control SSH and ICMP traffic.
- Validate private connectivity using ping and SSH.
- Verify that application instances do not require public IPv4 addresses.

## Architecture

```text
                         Internet
                            |
                     Internet Gateway
                            |
                 +----------------------+
                 |        VPC A         |
                 |   10.10.0.0/16       |
                 |                      |
                 |   Public Subnet      |
                 |   10.10.1.0/24       |
                 |                      |
                 |   Bastion Host       |
                 |   10.10.1.10         |
                 |          |           |
                 |   Private Subnet     |
                 |   10.10.2.0/24       |
                 |                      |
                 |   App-A              |
                 |   10.10.2.10         |
                 +----------+-----------+
                            |
                     VPC Peering
                            |
                 +----------+-----------+
                 |        VPC B         |
                 |   10.20.0.0/16       |
                 |                      |
                 |   Private Subnet     |
                 |   10.20.1.0/24       |
                 |                      |
                 |   App-B              |
                 |   10.20.1.10         |
                 +----------------------+
```

*Note: This is a logical architecture diagram. Confirm the deployed configuration and actual resource addresses before treating it as evidence of a successful deployment.*

## AWS Resources

| Resource | Name | Purpose |
|---|---|---|
| VPC A | `ewb-vpc-a` | Hosts the bastion and App-A |
| VPC B | `ewb-vpc-b` | Hosts App-B |
| Public subnet | `ewb-a-public` | Hosts the bastion instance |
| Private subnet A | `ewb-a-private` | Hosts App-A |
| Private subnet B | `ewb-b-private` | Hosts App-B |
| Internet Gateway | `ewb-igw-a` | Provides internet connectivity for eligible resources in VPC A |
| Public route table | `ewb-rt-a-public` | Routes public-subnet traffic |
| Private route table A | `ewb-rt-a-private` | Routes private-subnet traffic in VPC A |
| Private route table B | `ewb-rt-b-private` | Routes private-subnet traffic in VPC B |
| VPC Peering connection | `ewb-a-b-peer` | Connects VPC A and VPC B |
| Bastion security group | `ewb-sg-bastion` | Controls access to the bastion |
| App-A security group | `ewb-sg-app-a` | Controls access to App-A |
| App-B security group | `ewb-sg-app-b` | Controls access to App-B |
| Bastion EC2 | `ewb-bastion` | SSH entry point |
| Application EC2 | `ewb-app-a` | Application instance in VPC A |
| Application EC2 | `ewb-app-b` | Application instance in VPC B |

## IP Addressing Plan

AWS Region: `ap-south-1` (Asia Pacific — Mumbai)

| Network | CIDR Block | Role |
|---|---|---|
| VPC A | `10.10.0.0/16` | First VPC |
| Public subnet A | `10.10.1.0/24` | Bastion subnet |
| Private subnet A | `10.10.2.0/24` | App-A subnet |
| VPC B | `10.20.0.0/16` | Second VPC |
| Private subnet B | `10.20.1.0/24` | App-B subnet |

### EC2 Private IP Addresses

| Instance | Private IP | Public IPv4 |
|---|---|---|
| Bastion | `10.10.1.10` | Assigned if configured |
| App-A | `10.10.2.10` | None intended |
| App-B | `10.20.1.10` | None intended |

The two VPC CIDR blocks do not overlap, allowing them to be connected using VPC Peering.

## Network Configuration

### 1. Public Route Table

The public route table is associated with the public subnet in VPC A.

| Destination | Target |
|---|---|
| `10.10.0.0/16` | Local |
| `0.0.0.0/0` | Internet Gateway |
| `10.20.0.0/16` | VPC Peering connection |

### 2. Private Route Table — VPC A

Associated with the private subnet containing App-A.

| Destination | Target |
|---|---|
| `10.10.0.0/16` | Local |
| `10.20.0.0/16` | VPC Peering connection |

### 3. Private Route Table — VPC B

Associated with the private subnet containing App-B.

| Destination | Target |
|---|---|
| `10.20.0.0/16` | Local |
| `10.10.0.0/16` | VPC Peering connection |

**Important:** The VPC Peering connection must be active, and the relevant route tables must have the appropriate routes. Adding a peering connection alone does not automatically configure routing.

## Security Configuration

Security groups control which traffic can reach the EC2 instances.

### Bastion Security Group

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | Your public IP address `/32` | SSH access to bastion |

### App-A Security Group

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | Bastion private IP `10.10.1.10/32` | SSH access from bastion |
| ICMP Echo Request | N/A | App-B private IP `10.20.1.10/32` | Ping from App-B |

### App-B Security Group

| Protocol | Port | Source | Purpose |
|---|---:|---|---|
| TCP | 22 | Bastion private IP `10.10.1.10/32` | SSH access from bastion |
| ICMP Echo Request | N/A | App-A private IP `10.10.2.10/32` | Ping from App-A |

Review security group rules before deployment. Restrict SSH to trusted sources and allow only the traffic required for the lab.

## Implementation Steps

### Step 1: Create VPC A

- Create `ewb-vpc-a` with CIDR `10.10.0.0/16`.
- Create public subnet `ewb-a-public` with CIDR `10.10.1.0/24`.
- Create private subnet `ewb-a-private` with CIDR `10.10.2.0/24`.

### Step 2: Configure Internet Connectivity

- Create and attach `ewb-igw-a` to VPC A.
- Create `ewb-rt-a-public`.
- Add the default route `0.0.0.0/0` pointing to the Internet Gateway.
- Associate the public route table with the public subnet.

### Step 3: Create VPC B

- Create `ewb-vpc-b` with CIDR `10.20.0.0/16`.
- Create private subnet `ewb-b-private` with CIDR `10.20.1.0/24`.
- Create and associate `ewb-rt-b-private`.

### Step 4: Configure Security Groups

Create the bastion, App-A, and App-B security groups. Configure SSH and ICMP rules according to the security configuration above.

### Step 5: Launch EC2 Instances

Launch the following instances using Amazon Linux 2023, if that is the image selected for your deployment:

- `ewb-bastion` in the public subnet.
- `ewb-app-a` in the private subnet of VPC A.
- `ewb-app-b` in the private subnet of VPC B.

Use the `ewb-lab` key pair if that is the key pair configured in your deployment. Keep the application instances private.

### Step 6: Create VPC Peering

- Create a peering connection named `ewb-a-b-peer`.
- Select VPC A as the requester and VPC B as the accepter.
- Accept the peering request.
- Confirm the connection status is **Active**.

### Step 7: Configure Peering Routes

Add the appropriate peer-VPC CIDR route to each relevant route table:

- Public route table in VPC A: `10.20.0.0/16`
- Private route table in VPC A: `10.20.0.0/16`
- Private route table in VPC B: `10.10.0.0/16`

Select the VPC Peering connection as the target.

### Step 8: Validate Connectivity

Run the connectivity tests described below. Record actual command output and capture screenshots for the project report.

## Testing and Validation

Use the following checklist to document the results of your deployment.

| Test | Expected Result | Actual Result |
|---|---|---|
| Check peering status | Active | To be recorded |
| Ping App-B from App-A | Successful after routing and security rules are configured | To be recorded |
| Ping App-A from App-B | Successful after routing and security rules are configured | To be recorded |
| SSH to bastion | Successful with valid key and allowed source | To be recorded |
| SSH to App-A through bastion | Successful with correct key and security rules | To be recorded |
| SSH to App-B through bastion | Successful with correct key and security rules | To be recorded |
| Check App-A public IPv4 | No public IPv4 address | To be recorded |
| Check App-B public IPv4 | No public IPv4 address | To be recorded |
| Test restricted traffic | Blocked as intended | To be recorded |

### Example Connectivity Commands

From App-A, test App-B:

```bash
ping -c 4 10.20.1.10
```

From App-B, test App-A:

```bash
ping -c 4 10.10.2.10
```

SSH to a private instance through the bastion using a local SSH key:

```bash
ssh -i /path/to/ewb-lab.pem \
  -J ec2-user@<BASTION_PUBLIC_IP> \
  ec2-user@10.10.2.10
```

For App-B:

```bash
ssh -i /path/to/ewb-lab.pem \
  -J ec2-user@<BASTION_PUBLIC_IP> \
  ec2-user@10.20.1.10
```

Replace `<BASTION_PUBLIC_IP>` and `/path/to/ewb-lab.pem` with your actual values. Keep the private key secure and do not commit it to the repository.

**Troubleshooting:** If ping fails, check the peering status, route tables, security groups, network ACLs, instance IP addresses, and operating-system firewall. If SSH fails, verify key permissions, username, key pair, and security group rules.


## Security Best Practices

- Never commit private SSH keys, AWS access keys, passwords, or secrets.
- Do not use unrestricted SSH access (`0.0.0.0/0`) for the bastion.
- Keep application instances in private subnets without public IPv4 addresses.
- Allow only required protocols and ports in security groups.
- Use IAM roles and least-privilege permissions where applicable.
- Avoid including sensitive AWS account information in screenshots.
- Review and remove unused AWS resources after completing the lab.

## Cleanup

To avoid unnecessary AWS charges, remove lab resources when you no longer need them.

1. Terminate the EC2 instances.
2. Delete the VPC Peering connection.
3. Delete security groups created for the lab after they are no longer in use.
4. Delete custom route tables that are not main route tables and are no longer needed.
5. Detach and delete the Internet Gateway.
6. Delete the subnets.
7. Delete VPC A and VPC B.
8. Check for remaining EBS volumes, Elastic IP addresses, and other billable resources.

AWS may prevent deletion while dependencies remain. Remove dependent resources in the appropriate order and verify the final state in the AWS Console.

## Learning Outcomes

Through this project, the following AWS networking concepts can be practised:

- VPC and subnet design
- Public versus private subnets
- Internet Gateway routing
- Route table associations
- VPC Peering
- CIDR blocks and private IP addressing
- Security groups and network traffic filtering
- Bastion-host access patterns
- EC2 connectivity troubleshooting
- AWS resource lifecycle management
