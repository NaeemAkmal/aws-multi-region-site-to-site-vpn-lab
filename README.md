# AWS Multi-Region Site-to-Site VPN Lab

**AWS | Cloud Infrastructure | Network Security | IKEv2/IPsec | Libreswan | Linux**

A practical implementation of secure, encrypted network connectivity between two Virtual Private Clouds (VPCs) deployed across different AWS Regions.

This project demonstrates the design, deployment, configuration, troubleshooting, and verification of an AWS Site-to-Site VPN using an AWS-managed Virtual Private Gateway and an EC2-based Customer Gateway running **Libreswan 4.12 on Amazon Linux 2023**.

The implementation connects resources in the Mumbai (`ap-south-1`) and N. Virginia (`us-east-1`) Regions using IKEv2 and IPsec. The completed lab successfully demonstrated private EC2 connectivity across Regions, established IPsec security associations, and bidirectional encrypted traffic.

Beyond deployment, the project documents practical troubleshooting involving VPN endpoint configuration, authentication failures, IPsec negotiation, and network routing.

## Implementation Summary

| Category | Implementation |
|---|---|
| Cloud Provider | Amazon Web Services |
| AWS Regions | Mumbai and N. Virginia |
| VPCs | 2 |
| EC2 Instances | 2 |
| Connectivity | AWS Site-to-Site VPN |
| AWS VPN Endpoint | Virtual Private Gateway |
| Customer VPN Endpoint | EC2 running Libreswan 4.12 |
| VPN Protocol | IKEv2 / IPsec |
| Authentication | Pre-Shared Key |
| Encryption | AES-256 |
| Integrity | SHA-256 |
| Routing | Static |
| Operational Tunnel | Tunnel 1 |
| Connectivity Test | 4/4 ICMP replies |
| Packet Loss | 0% |
| Encrypted Traffic | Verified |
| Final Status | Successfully completed and tested |

**Deployment status:** This is a documented historical lab. The AWS resources were removed after testing.

---

## Table of Contents

1. [Network Architecture](#network-architecture)
2. [AWS Infrastructure Inventory](#aws-infrastructure-inventory)
3. [Network Design and IP Addressing](#network-design-and-ip-addressing)
4. [Implementation Guide](#implementation-guide)
   - [VPC Deployment](#phase-1-vpc-deployment)
   - [Subnet Configuration](#phase-2-subnet-configuration)
   - [EC2 Instance Deployment](#phase-3-ec2-instance-deployment)
   - [Security Groups and Network Controls](#phase-4-security-groups-and-network-controls)
   - [Route Table Configuration](#phase-5-route-table-configuration)
   - [VPN Gateway Deployment](#phase-6-vpn-gateway-deployment)
   - [Libreswan Configuration](#phase-7-libreswan-installation-and-configuration)
   - [VPN Authentication and Tunnel Establishment](#phase-8-vpn-authentication-and-tunnel-establishment)
   - [Static VPN Routing](#phase-9-static-vpn-routing)
5. [Troubleshooting and Root Cause Analysis](#troubleshooting-and-root-cause-analysis)
6. [Testing and Verification](#testing-and-verification)
7. [Evidence and Screenshots](#evidence-and-screenshots)
8. [Security Considerations](#security-considerations)
9. [Final Results](#final-results)
10. [Technical Lessons Learned](#technical-lessons-learned)
11. [Limitations and Future Improvements](#limitations-and-future-improvements)
12. [Conclusion](#conclusion)

---

## Network Architecture

![AWS Multi-Region Site-to-Site VPN Architecture](images/00-architecture-diagram.png)

*Figure 1: Logical architecture of the Site-to-Site VPN connecting the Mumbai and N. Virginia AWS environments.*

The infrastructure consists of two VPC networks deployed in different AWS Regions.

### Mumbai — AWS-Managed VPN Side

The Mumbai environment contains:

- VPC-A with CIDR `10.100.0.0/16`.
- A private subnet hosting EC2-A.
- An EC2 instance with private IP `10.100.0.190`.
- A Virtual Private Gateway attached to VPC-A.
- A private route table directing traffic for the remote VPC toward the Virtual Private Gateway.

### N. Virginia — Customer-Managed VPN Side

The N. Virginia environment contains:

- VPC-B with CIDR `10.200.0.0/16`.
- A public subnet for the customer-managed VPN instance.
- EC2-B with private IP `10.200.0.206`.
- An Elastic IP associated with the EC2-based VPN endpoint.
- Amazon Linux 2023.
- Libreswan 4.12.
- A Customer Gateway resource representing the EC2 VPN endpoint.

### Logical Traffic Flow

```text
                     AWS CLOUD

        N. Virginia (us-east-1)
        ------------------------
        VPC-B: 10.200.0.0/16
                   |
          Public Subnet
                   |
           EC2-B Instance
          10.200.0.206
           Amazon Linux
          Libreswan 4.12
                   |
           Customer Gateway
                   |
                   |
           IKEv2 + IPsec ESP
          Encrypted VPN Tunnel
                Tunnel 1
                   |
                   |
          Virtual Private Gateway
                   |
         VPC-A Private Routing
                   |
            Private Subnet
                   |
           EC2-A Instance
          10.100.0.190
                   |
          VPC-A: 10.100.0.0/16
        ------------------------
             Mumbai (ap-south-1)
```

The tested traffic originated from the EC2-based VPN endpoint in N. Virginia and was delivered to the private EC2 instance in Mumbai.

The IPsec tunnel protected communication between the two configured network ranges.

The implementation verified connectivity between these two EC2 instances. Connectivity for additional hosts throughout both VPCs was not separately tested.

---

## AWS Infrastructure Inventory

### Regional Infrastructure

| Resource | Mumbai | N. Virginia |
|---|---|---|
| Region Code | `ap-south-1` | `us-east-1` |
| VPC Name | VPC-A | VPC-B |
| VPC CIDR | `10.100.0.0/16` | `10.200.0.0/16` |
| Subnet Type | Private | Public |
| EC2 Instance | EC2-A | EC2-B |
| Private IPv4 | `10.100.0.190` | `10.200.0.206` |
| VPN Role | Destination network | Customer-managed gateway |
| VPN Component | Virtual Private Gateway | Customer Gateway |
| VPN Implementation | AWS-managed | Libreswan 4.12 |
| Operating System | Not recorded | Amazon Linux 2023 |
| Public Address | Not required for tested destination | Elastic IP assigned |

### Networking Components

| Component | Purpose |
|---|---|
| VPC | Provides isolated AWS networking environments |
| Subnet | Defines address ranges and placement of EC2 instances |
| Route Table | Determines how network traffic is forwarded |
| Internet Gateway | Provides internet connectivity for an appropriately routed public subnet |
| Security Group | Controls instance-level inbound and outbound traffic |
| Network ACL | Provides subnet-level traffic filtering |
| Elastic IP | Provides a stable public endpoint for the customer gateway |
| Virtual Private Gateway | AWS-managed VPN termination point |
| Customer Gateway | Defines the external, EC2-based VPN endpoint |
| Site-to-Site VPN | Establishes encrypted connectivity |
| Libreswan | Manages IKE and IPsec on the Linux gateway |

---

## Network Design and IP Addressing

The network was designed using non-overlapping private IPv4 address ranges.

This separation was necessary to avoid ambiguity when forwarding traffic between the two VPC environments.

### IPv4 Addressing Plan

| Network | CIDR / Address | Description |
|---|---|---|
| Mumbai VPC-A | `10.100.0.0/16` | Destination VPC |
| Mumbai private subnet | Exact subnet CIDR not separately recorded | Hosts EC2-A |
| Mumbai EC2-A | `10.100.0.190` | Private test destination |
| N. Virginia VPC-B | `10.200.0.0/16` | Customer gateway VPC |
| N. Virginia public subnet | `10.200.0.0/24` | Hosts EC2-B |
| N. Virginia EC2-B | `10.200.0.206` | Linux VPN endpoint |
| Customer Gateway Public IP | Omitted from published documentation | Public VPN endpoint |
| AWS VPN Tunnel Public IP | Omitted from published documentation | AWS-managed peer |

### Addressing Design

The use of separate VPC address spaces enables the routing infrastructure to distinguish local traffic from traffic intended for the remote network.

The two primary network ranges were included in the Libreswan connection configuration as the IPsec traffic selectors.

**Local network:**

`10.200.0.0/16`

**Remote network:**

`10.100.0.0/16`

---

# Implementation Guide

The following sections explain the infrastructure design and configuration in deployment order, followed by the actual troubleshooting and verification results.

## Phase 1: VPC Deployment

### Step 1.1: Create VPC-A in Mumbai

The first VPC was deployed in the Mumbai Region.

**Configuration:**

| Property | Value |
|---|---|
| VPC | VPC-A |
| Region | Mumbai |
| Region Code | `ap-south-1` |
| IPv4 CIDR | `10.100.0.0/16` |
| Purpose | Host the destination private EC2 workload |

The network provided the AWS-side environment for testing private communication through the VPN.

![Mumbai VPC](images/01-mumbai-vpc.png)

*Figure 2: VPC-A in the Mumbai Region using CIDR 10.100.0.0/16.*

### Step 1.2: Create VPC-B in N. Virginia

The second VPC was deployed in the N. Virginia Region.

**Configuration:**

| Property | Value |
|---|---|
| VPC | VPC-B |
| Region | N. Virginia |
| Region Code | `us-east-1` |
| IPv4 CIDR | `10.200.0.0/16` |
| Purpose | Host the Linux-based VPN endpoint |

VPC-B provided the network in which the customer-managed VPN gateway was deployed.

![N. Virginia VPC](images/02-virginia-vpc.png)

*Figure 3: VPC-B in the N. Virginia Region using CIDR 10.200.0.0/16.*

### Design Considerations

Both VPCs use private IPv4 ranges.

Their CIDR blocks do not overlap.

The separate AWS Regions provide geographic separation, while the VPN allows selected private traffic to traverse the encrypted connection between the environments.

---

## Phase 2: Subnet Configuration

Subnets were used to organize EC2 resources within the VPCs.

### Step 2.1: Mumbai Private Subnet

The destination EC2 instance was placed in a private subnet within VPC-A.

| Property | Configuration |
|---|---|
| VPC | VPC-A |
| Subnet Type | Private |
| Parent CIDR | `10.100.0.0/16` |
| Assigned EC2 IP | `10.100.0.190` |
| Routing Requirement | Remote VPC traffic through Virtual Private Gateway |

The exact subnet CIDR was not preserved in the available evidence, so it is not reconstructed here.

The private subnet was associated with a route table containing the remote network route.

### Step 2.2: N. Virginia Public Subnet

The VPN endpoint instance was deployed within a public subnet in VPC-B.

| Property | Configuration |
|---|---|
| VPC | VPC-B |
| Subnet Type | Public |
| Subnet CIDR | `10.200.0.0/24` |
| EC2 Private IP | `10.200.0.206` |
| Public Endpoint | Elastic IP |
| Purpose | Host the customer-managed VPN gateway |

The VPN instance required public network connectivity so that its IKE/IPsec endpoint could communicate with the AWS-managed VPN endpoint.

A public subnet generally requires a route toward an Internet Gateway for public internet access. The specific historical Internet Gateway and route table identifiers were not preserved in this project record.

### Why Separate Public and Private Subnets?

The architecture demonstrates two different networking requirements:

- A private destination workload that communicates through a VPN.
- A publicly reachable VPN endpoint that negotiates and maintains the encrypted tunnel.

The private destination does not require its own public IP address for the tested VPN communication.

---

## Phase 3: EC2 Instance Deployment

Two EC2 instances were used for the implementation.

### Step 3.1: Deploy EC2-A in Mumbai

EC2-A functioned as the private destination for VPN connectivity testing.

| Property | Value |
|---|---|
| Instance | EC2-A |
| AWS Region | Mumbai |
| VPC | VPC-A |
| Subnet | Private |
| Private IPv4 | `10.100.0.190` |
| VPN Software | Not required on destination instance |
| Purpose | Receive private traffic through the AWS VPN |

This instance was used as the endpoint for the ICMP connectivity test.

The instance was reachable through its private address after the VPN tunnel and routing were correctly configured.

### Step 3.2: Deploy EC2-B in N. Virginia

EC2-B acted as the Linux-based VPN endpoint.

| Property | Value |
|---|---|
| Instance | EC2-B |
| AWS Region | N. Virginia |
| VPC | VPC-B |
| Subnet | Public |
| Private IPv4 | `10.200.0.206` |
| Public Endpoint | Elastic IP |
| Operating System | Amazon Linux 2023 |
| VPN Software | Libreswan 4.12 |
| Primary Role | Customer-managed IPsec gateway |

The instance provided the operating environment for Libreswan, including IKE negotiation and Linux kernel IPsec processing.

### Step 3.3: Configure Public Connectivity

An Elastic IP was used for the customer gateway endpoint.

The Elastic IP allowed the AWS Site-to-Site VPN configuration to reference the public-facing address of the Linux VPN instance.

The connection configuration also required the customer gateway's public identity to match the peer identity used by Libreswan.

### Step 3.4: Instance Routing Considerations

An EC2 instance performing transit forwarding for other workloads may require AWS source/destination checks to be disabled and additional VPC routes to its network interface.

Those requirements are relevant to a broader network-appliance deployment.

In this lab, the verified test traffic originated from the Libreswan EC2 instance itself. Source/destination-check settings and forwarding for additional VPC-B hosts were not separately verified.

---

## Phase 4: Security Groups and Network Controls

Security Groups are stateful firewalls attached to EC2 network interfaces.

They control which traffic is permitted to enter and leave an instance.

The VPN architecture requires two distinct categories of traffic:

1. Public-side IKE/IPsec communication between the Linux customer gateway and AWS.
2. Private-side application or diagnostic traffic between the connected networks.

### Security Group Design

| Instance | Security Group Role | Purpose |
|---|---|---|
| EC2-A | Private workload access | Allow authorized VPN-originated test traffic |
| EC2-B | VPN gateway access | Permit IKE/IPsec peer communication and required administration |

### EC2-B: Customer Gateway Security Group

The following table describes the traffic requirements for the design. It is a reference for verifying or rebuilding the environment, not an exact export of the original Security Group.

| Direction | Protocol | Port / Type | Peer or Destination | Purpose |
|---|---|---|---|---|
| Inbound | UDP | 500 | AWS VPN peer | IKE negotiation |
| Inbound | UDP | 4500 | AWS VPN peer | IPsec NAT Traversal |
| Outbound | UDP | 500 | AWS VPN peer | IKE communication |
| Outbound | UDP | 4500 | AWS VPN peer | NAT-T traffic |
| Outbound | ICMP | Echo Request | Authorized private destination | Connectivity testing |
| Inbound | TCP | 22, if SSH is used | Trusted administrator IP only | Restricted management |

ESP protocol 50 must also be considered when IPsec ESP is not encapsulated in UDP. This lab used NAT Traversal settings.

### EC2-A: Private Workload Security Group

To perform the verified ICMP test, the destination instance needed to permit the appropriate traffic.

A least-privilege design for reproducing the test is:

| Direction | Protocol | Source / Destination | Purpose |
|---|---|---|---|
| Inbound | ICMP | Authorized remote VPN network or specific VPN source | Permit echo requests |
| Outbound | As required | Authorized destinations | Return or application traffic |
| Management | Only if required | Trusted administration source | Controlled instance access |

Security Groups are stateful, so permitted inbound request traffic can receive associated responses without a separate matching outbound rule, subject to the applicable AWS behavior and network controls.

The precise historical rule IDs, source ranges, and rule descriptions were not captured.

### Network ACL Considerations

Network ACLs operate at the subnet level and are stateless.

A deployment must account for traffic in both directions, including any required response traffic.

The available project evidence does not contain an export of the original Network ACL rules.

### Security Principles

The design illustrates several important infrastructure-security considerations:

- Limit public management access to trusted sources.
- Permit VPN negotiation only from expected endpoints where practical.
- Allow private test traffic only between intended networks.
- Avoid unnecessary public exposure of the destination EC2 instance.
- Keep authentication material separate from public documentation.
- Review both instance-level and subnet-level filtering during troubleshooting.

---

## Phase 5: Route Table Configuration

Route tables determine how traffic leaves a subnet.

For private VPN connectivity, route entries must direct packets toward the correct network destination.

### Step 5.1: Mumbai VPC-A Routing

The Mumbai private subnet route table was configured with two verified routes:

| Destination | Target | State |
|---|---|---|
| `10.100.0.0/16` | Local | Active |
| `10.200.0.0/16` | Virtual Private Gateway | Active |

The local route allows communication within VPC-A.

The second route directs packets intended for the N. Virginia network toward the AWS Virtual Private Gateway.

![Mumbai Private Route Table](images/08-mumbai-private-route-table.png)

*Figure 4: Mumbai private route table showing the local VPC route and remote VPN destination.*

### Step 5.2: N. Virginia Routing

The N. Virginia public subnet required connectivity to the public AWS VPN endpoint.

Its baseline routing design included:

- A local route for VPC-B.
- Public internet connectivity for the customer gateway instance.
- Linux IPsec traffic selectors and routing behavior for the remote Mumbai network.

The exact original VPC-B route table entries were not archived as independent evidence.

For the completed test, Libreswan on EC2-B successfully processed traffic destined for the Mumbai private address through the established IPsec tunnel.

### Routing Considerations

Correct routing must be considered in both directions.

The VPN connection can report an established state while traffic remains unsuccessful because of:

- Missing remote network routes.
- Incorrect route table associations.
- Incorrect VPN route configuration.
- Firewall or Security Group rules.
- Incorrect IPsec traffic selectors.
- Operating-system routing behavior.

The lab encountered and resolved an AWS VPN static routing issue.

---

## Phase 6: VPN Gateway Deployment

### Step 6.1: Create the Virtual Private Gateway

An AWS Virtual Private Gateway was created for VPC-A in Mumbai.

The Virtual Private Gateway acts as the AWS-managed endpoint for the VPN connection.

It was attached to the Mumbai VPC.

**Purpose:**

- Terminate the AWS side of the VPN.
- Enable private traffic between the VPN and VPC-A.
- Participate in routing traffic to the remote network.

### Step 6.2: Create the Customer Gateway

A Customer Gateway resource was created in AWS to represent the customer-managed VPN endpoint.

The resource referenced the public IP address associated with EC2-B in N. Virginia.

**Customer Gateway Details:**

| Parameter | Configuration |
|---|---|
| Resource Type | AWS Customer Gateway |
| VPN Device | EC2-B |
| Operating System | Amazon Linux 2023 |
| VPN Software | Libreswan 4.12 |
| Public Endpoint | Elastic IP |
| Routing Method | Static |

The Customer Gateway resource represents the VPN peer. The VPN software itself runs on the Linux EC2 instance.

### Step 6.3: Create the Site-to-Site VPN Connection

The AWS Site-to-Site VPN connection was configured using:

- The Mumbai Virtual Private Gateway.
- The N. Virginia Customer Gateway.
- Static routing.
- Pre-shared-key authentication.
- IKEv2-compatible tunnel settings.

AWS provisioned two tunnel endpoints for the VPN connection.

The lab configured and verified Tunnel 1.

### Step 6.4: Verify the AWS Tunnel State

Following successful configuration and authentication, AWS reported Tunnel 1 as UP.

![AWS VPN Tunnel Status](images/03-vpn-tunnel-up.png)

*Figure 5: AWS Site-to-Site VPN connection showing the operational first tunnel.*

**Observed result:**

| Tunnel | Result |
|---|---|
| Tunnel 1 | UP |
| Tunnel 2 | DOWN / Not configured |

The second tunnel was not configured or tested for automatic failover.

---

## Phase 7: Libreswan Installation and Configuration

### Step 7.1: Install Libreswan

The Linux VPN endpoint used Amazon Linux 2023 with Libreswan 4.12.

Libreswan was installed using the system package manager.

```bash
sudo dnf install -y libreswan
```

### Step 7.2: Enable the IPsec Service

The IPsec service was configured to start automatically.

```bash
sudo systemctl enable ipsec
sudo systemctl start ipsec
```

Service status could be checked with:

```bash
sudo systemctl status ipsec --no-pager
```

### Step 7.3: Configure IP Forwarding

Linux IP forwarding was enabled.

Persistent kernel settings were stored under `/etc/sysctl.d/`.

Example configuration:

```ini
net.ipv4.ip_forward=1
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.all.send_redirects=0
```

Apply the configuration:

```bash
sudo sysctl --system
```

IP forwarding is especially important when the VPN instance forwards packets on behalf of other hosts. It does not replace the need for correct AWS routing and instance-level controls.

### Step 7.4: Configure the IPsec Tunnel

Libreswan connection settings were stored in:

`/etc/ipsec.d/aws.conf`

The following sanitized example documents the tunnel parameters used during implementation.

```ini
conn Tunnel1
    authby=secret
    auto=start
    type=tunnel
    ikev2=insist

    left=%defaultroute
    leftid=<CUSTOMER_GATEWAY_PUBLIC_IP>
    leftsubnet=10.200.0.0/16

    right=<AWS_TUNNEL_OUTSIDE_IP>
    rightsubnet=10.100.0.0/16

    ike=aes256-sha2_256;modp2048
    phase2alg=aes256-sha2_256;modp2048

    ikelifetime=8h
    salifetime=1h

    encapsulation=yes

    dpddelay=30
    dpdaction=restart
```

**Configuration Parameters**

| Parameter | Description |
|---|---|
| `conn Tunnel1` | Defines the VPN connection |
| `authby=secret` | Uses pre-shared-key authentication |
| `auto=start` | Automatically initiates the connection |
| `type=tunnel` | Configures tunnel-mode IPsec |
| `ikev2=insist` | Requires IKEv2 negotiation |
| `left=%defaultroute` | Selects the local default-route interface |
| `leftid` | Defines the local VPN identity |
| `leftsubnet` | Specifies the N. Virginia network |
| `right` | Defines the remote AWS VPN endpoint |
| `rightsubnet` | Specifies the Mumbai network |
| `ike` | Defines IKE cryptographic settings |
| `phase2alg` | Defines IPsec ESP settings |
| `ikelifetime` | Sets the IKE SA lifetime |
| `salifetime` | Sets the IPsec SA lifetime |
| `encapsulation=yes` | Enables UDP encapsulation behavior |
| `dpddelay` | Configures dead-peer detection interval |
| `dpdaction=restart` | Requests re-establishment after peer failure |

The public endpoints have been replaced with placeholders. This configuration is included as a technical reference rather than an immediately deployable file.

### Step 7.5: Configure Pre-Shared-Key Authentication

The VPN used pre-shared-key authentication.

Libreswan stored the authentication secret separately in:

`/etc/ipsec.secrets`

The file was restricted to root access.

```bash
sudo chown root:root /etc/ipsec.secrets
sudo chmod 600 /etc/ipsec.secrets
```

Correct peer identity mapping was essential for successful authentication.

The project does not publish the actual pre-shared keys.

---

## Phase 8: VPN Authentication and Tunnel Establishment

After completing the AWS gateway and Libreswan configuration, the VPN connection was initiated.

The connection process involved two main security stages.

### Stage 1: IKE Security Association

IKEv2 performs negotiation and authentication between the VPN peers.

Successful completion results in an established IKE Security Association.

### Stage 2: IPsec Child Security Association

The peers negotiate an IPsec Child Security Association used to protect traffic matching the configured selectors.

The completed connection was verified using:

```bash
sudo ipsec status
```

The final observed state included:

```text
Total IPsec connections: loaded 1, active 1

STATE_V2_ESTABLISHED_IKE_SA
STATE_V2_ESTABLISHED_CHILD_SA
```

![Established IPsec Connection](images/05-ipsec-established.png)

*Figure 6: Libreswan status showing successful IKE and IPsec Child Security Association establishment.*

These states confirmed that the security associations were established.

Additional connectivity testing was still necessary to confirm that private network traffic was actually working.

---

## Phase 9: Static VPN Routing

### Step 9.1: Configure the Remote Network Route

The VPN connection used static routing.

The remote N. Virginia network was added to the AWS VPN connection:

```text
Remote Network: 10.200.0.0/16
```

![VPN Static Route](images/04-static-route.png)

*Figure 7: Static routing configuration for the AWS Site-to-Site VPN connection.*

### Step 9.2: Verify VPC-A Routing

The Mumbai private route table included a route to the N. Virginia network through the Virtual Private Gateway.

This allowed traffic returning toward the customer gateway network to follow the VPN path.

### Step 9.3: Validate End-to-End Routing

After correcting the AWS VPN static route, the N. Virginia EC2 instance could communicate with the Mumbai EC2 instance.

This was the final configuration change that resolved the observed private connectivity failure.

---

# Troubleshooting and Root Cause Analysis

Troubleshooting was a major part of the project and provided practical experience in identifying problems across multiple networking layers.

The implementation required investigation of IKE negotiation, authentication identities, Libreswan configuration loading, AWS routing, and IPsec traffic counters.

## Troubleshooting Summary

| Issue | Symptoms | Root Cause | Resolution |
|---|---|---|---|
| Incorrect VPN Endpoint | IKEv2 retransmissions and no response | Incorrect AWS tunnel IP | Corrected remote endpoint |
| PSK Authentication Failure | `no PSK found` | Incorrect peer identity mapping | Corrected secrets configuration |
| Duplicate Configuration | Repeated include directive | Duplicate Libreswan include statement | Removed duplicate |
| Private Connectivity Failure | VPN UP but ping failed | Missing AWS VPN static route | Added `10.200.0.0/16` |
| No New ESP Traffic | Initial counters unchanged | No test traffic on current SA | Generated traffic and checked counters |

## Incident 1: IKEv2 Negotiation Timeout

**Observed Problem**

The VPN connection initially failed to establish.

Libreswan attempted to initiate IKEv2 negotiation, but the remote endpoint did not respond as expected.

Diagnostic output included:

```text
STATE_V2_PARENT_I1
RETRANSMIT
No response (or no acceptable response)
```

**Investigation**

The connection parameters were reviewed, including the configured remote tunnel endpoint.

IKE traffic and Libreswan logs were examined to determine whether the AWS VPN service was responding.

**Root Cause**

The remote IP address in the Libreswan configuration did not match the intended AWS VPN tunnel endpoint.

**Resolution**

The endpoint address was corrected using the AWS VPN configuration details.

After the correction, IKE negotiation progressed.

**Lesson Learned**

Before troubleshooting advanced cryptographic settings, verify basic network reachability and peer endpoint information.

---

## Incident 2: VPN Authentication Failure

**Observed Problem**

After the endpoint issue was corrected, the negotiation advanced but authentication failed.

Libreswan reported:

```text
authentication failed: no PSK found
Failed to find our PreShared Key
```

**Investigation**

The IPsec peer identities and the entries in `/etc/ipsec.secrets` were compared.

**Root Cause**

The configured pre-shared-key identity mapping did not match the identities expected during IKE authentication.

**Resolution**

The secrets file was corrected to reference the appropriate VPN peer identities.

The IPsec service was restarted, and authentication succeeded.

**Lesson Learned**

A valid pre-shared key is not sufficient if the VPN peer identities do not match the corresponding secret configuration.

---

## Incident 3: Duplicate Libreswan Configuration

**Observed Problem**

A repeated include statement was identified during configuration review.

**Investigation**

The primary Libreswan configuration file and included configuration paths were reviewed.

**Root Cause**

The same configuration directory was included more than once.

**Resolution**

The repeated include directive was removed.

**Lesson Learned**

Configuration consistency matters when troubleshooting networking services. Duplicate entries can complicate investigation and should be removed.

---

## Incident 4: Tunnel UP but Private Connectivity Failed

**Observed Problem**

The VPN tunnel showed an established connection, but private communication between the EC2 instances initially failed.

The successful tunnel status did not translate into successful end-to-end network connectivity.

**Investigation**

The following areas were checked:

- AWS VPN tunnel state.
- IKE and IPsec security associations.
- VPN connection static routes.
- Mumbai private route table.
- Remote CIDR configuration.
- Private connectivity using ICMP.

**Root Cause**

The AWS Site-to-Site VPN connection was missing the required static route for the N. Virginia network.

**Resolution**

The missing static route was added:

```text
10.200.0.0/16
```

The route table association and routing behavior were subsequently verified.

After this correction, private communication succeeded.

**Lesson Learned**

A VPN connection can be cryptographically established while application or network communication still fails.

Successful VPN implementation requires both secure tunnel establishment and correct routing.

---

## Incident 5: IPsec Traffic Counters Initially Unchanged

**Observed Problem**

The newly established IPsec security association initially showed no traffic activity.

**Investigation**

The IPsec connection state and traffic counters were examined.

**Resolution**

ICMP traffic was generated from the N. Virginia EC2 instance toward the Mumbai private EC2 instance.

The counters were checked again and showed increased inbound and outbound traffic.

**Lesson Learned**

VPN status and packet counters should be evaluated together. Establishing a tunnel does not prove that application traffic is being processed.

---

# Testing and Verification

Three verification methods were used to establish that the VPN was functioning.

## Test 1: VPN and IPsec State Verification

**Objective**

Confirm successful IKEv2 negotiation and IPsec Child Security Association establishment.

**Command**

```bash
sudo ipsec status
```

**Observed Results**

| Check | Result |
|---|---|
| IPsec Service | Operational |
| Active Connection | 1 |
| IKE Security Association | Established |
| IPsec Child Security Association | Established |
| AWS Tunnel 1 | UP |

![IPsec Established](images/05-ipsec-established.png)

**Test Result: Passed**

The connection was successfully negotiated and authenticated.

---

## Test 2: Private EC2-to-EC2 Connectivity

**Objective**

Verify that EC2-B in N. Virginia could reach EC2-A in Mumbai using the destination's private IPv4 address.

**Command**

```bash
ping -c 4 -W 2 10.100.0.190
```

**Observed Results**

| Metric | Value |
|---|---|
| Source | EC2-B |
| Source Region | N. Virginia |
| Destination | EC2-A |
| Destination Region | Mumbai |
| Destination Private IP | `10.100.0.190` |
| Packets Transmitted | 4 |
| Packets Received | 4 |
| Packet Loss | 0% |
| Average Round-Trip Time | Approximately 200 ms |

![Successful Ping Test](images/06-successful-ping.png)

*Figure 8: Successful private EC2 connectivity across AWS Regions.*

**Test Result: Passed**

The ICMP results demonstrated working private connectivity between the tested EC2 instances.

The measured latency represents the observed test, not a guaranteed performance level.

---

## Test 3: IPsec Encrypted Traffic Verification

**Objective**

Verify that traffic was processed through the established IPsec connection.

**Command**

```bash
sudo ipsec trafficstatus
```

The Linux IPsec counters were examined before and after generating network traffic.

**Observed Counter Changes**

| Traffic Direction | Before | After | Change |
|---|---|---|---|
| ESP Inbound | 336 bytes | 672 bytes | +336 bytes |
| ESP Outbound | 336 bytes | 672 bytes | +336 bytes |

![IPsec Traffic Counters](images/07-encrypted-traffic.png)

*Figure 9: Increased inbound and outbound IPsec traffic counters following the connectivity test.*

**Test Result: Passed**

The observed counter increases provided evidence that traffic was processed by the IPsec security association.

---

# Evidence and Screenshots

The repository includes the following implementation screenshots.

| File | Evidence |
|---|---|
| `00-architecture-diagram.png` | Complete AWS VPN architecture |
| `01-mumbai-vpc.png` | Mumbai VPC configuration |
| `02-virginia-vpc.png` | N. Virginia VPC configuration |
| `03-vpn-tunnel-up.png` | Operational VPN Tunnel 1 |
| `04-static-route.png` | VPN static route configuration |
| `05-ipsec-established.png` | Established IKE and IPsec security associations |
| `06-successful-ping.png` | Successful private network connectivity |
| `07-encrypted-traffic.png` | Increasing IPsec traffic counters |
| `08-mumbai-private-route-table.png` | Mumbai private route table |

### Repository Structure

```text
aws-multi-region-site-to-site-vpn-lab/
|
|-- README.md
|
`-- images/
    |
    |-- 00-architecture-diagram.png
    |-- 01-mumbai-vpc.png
    |-- 02-virginia-vpc.png
    |-- 03-vpn-tunnel-up.png
    |-- 04-static-route.png
    |-- 05-ipsec-established.png
    |-- 06-successful-ping.png
    |-- 07-encrypted-traffic.png
    `-- 08-mumbai-private-route-table.png
```

The README contains the project documentation, while the `images/` directory contains supporting visual evidence.

---

# Security Considerations

The project involved several security considerations relevant to cloud networking and VPN deployments.

### VPN Authentication

The tunnel used pre-shared-key authentication.

Actual authentication secrets are not included in the configuration examples.

### Encrypted Communication

IKEv2 and IPsec were used to establish and protect the VPN connection.

The implemented cryptographic parameters included AES-256 and SHA-256.

### Network Isolation

The destination workload was hosted in a private subnet.

Its verified VPN connectivity used private addressing rather than requiring a public destination IP.

### Access Control

Security Groups and subnet-level network policies must permit the appropriate VPN and private traffic while restricting unnecessary access.

### Operational Security

The documentation replaces the public VPN peer addresses with placeholders and does not intentionally publish live credentials.

Historical screenshots and Git commit history should also be reviewed for sensitive information before redistribution.

### Resource Lifecycle

The AWS resources were removed after the lab was completed.

This prevents the documentation from being mistaken for a currently running or monitored production environment.

---

# Final Results

| Component | Outcome |
|---|---|
| Multi-Region VPC Deployment | Completed |
| Mumbai VPC-A | Configured |
| N. Virginia VPC-B | Configured |
| EC2-A Private Workload | Deployed |
| EC2-B VPN Endpoint | Deployed |
| Amazon Linux 2023 VPN Host | Configured |
| Libreswan 4.12 | Operational |
| Virtual Private Gateway | Configured |
| Customer Gateway | Configured |
| Site-to-Site VPN | Established |
| Tunnel 1 | UP |
| IKEv2 Authentication | Successful |
| IPsec Child Security Association | Established |
| AWS VPN Static Route | Corrected and Verified |
| Mumbai Private Route Table | Verified |
| Private EC2 Connectivity | Successful |
| ICMP Packet Loss | 0% |
| Bidirectional IPsec Traffic | Verified |
| Tunnel 2 High Availability | Not implemented |
| AWS Lab Cleanup | Completed |

**Final outcome:** An encrypted Site-to-Site VPN connection was successfully established between the Linux VPN gateway in N. Virginia and the private AWS network in Mumbai.

The implementation was validated through VPN state inspection, private EC2 connectivity testing, and increasing inbound and outbound IPsec traffic counters.

---

# Technical Lessons Learned

### 1. Cloud Architecture

The lab provided practical experience with VPC network isolation, AWS gateway components, subnet design, and cross-region networking.

### 2. Linux Administration

Installing, configuring, and troubleshooting Libreswan strengthened hands-on Linux networking and service-management skills.

### 3. VPN Technologies

The implementation demonstrated the relationship between IKEv2 authentication, IPsec security associations, encryption settings, and protected traffic.

### 4. Network Routing

The static routing issue demonstrated that VPN establishment and end-to-end connectivity must be validated independently.

### 5. Technical Troubleshooting

The project reinforced a systematic troubleshooting approach:

```text
Review Configuration
        |
Verify Endpoint Connectivity
        |
Check IKE Authentication
        |
Confirm IPsec Security Associations
        |
Inspect VPN and VPC Routes
        |
Test Private Connectivity
        |
Verify Encrypted Traffic Counters
```

### 6. Documentation and Verification

The project emphasized the importance of preserving architecture diagrams, implementation screenshots, command outputs, and explanations of the issues encountered.

Rather than relying only on an AWS tunnel status indicator, multiple tests were used to confirm the result.

---

# Limitations and Future Improvements

The implementation was a functional learning lab, not a fully production-hardened environment.

The following areas were not part of the verified implementation:

| Area | Current Scope | Future Improvement |
|---|---|---|
| VPN Availability | Single active tunnel | Configure the second tunnel |
| Tunnel Failover | Not tested | Validate automatic failover |
| Routing | Static | Evaluate dynamic routing with BGP |
| Monitoring | Manual verification | Add monitoring and alerts |
| Workload Connectivity | EC2-B to EC2-A tested | Validate additional hosts and applications |
| Performance | Basic ICMP test | Perform throughput and stability testing |
| Infrastructure Automation | Manual deployment | Explore reproducible Infrastructure as Code |
| Security Validation | Functional VPN tests | Add a documented security configuration review |

Further improvements would require a new deployment and additional validation.

---

# Conclusion

This project demonstrated the implementation of an AWS Site-to-Site VPN between two VPC environments deployed in Mumbai and N. Virginia.

The infrastructure combined AWS networking services, an EC2-based Libreswan customer gateway, IKEv2 authentication, IPsec encryption, and static routing.

The project successfully established a VPN tunnel and verified private connectivity between the two test instances.

More importantly, the implementation involved resolving several real configuration problems, including an incorrect VPN endpoint, an authentication identity mismatch, and a missing static VPN route.

These challenges provided practical experience in AWS infrastructure, Linux administration, routing, network security, and structured troubleshooting.

The final results were supported by AWS configuration screenshots, established IPsec security associations, successful ICMP connectivity, and increasing encrypted traffic counters.

---

## References

- [AWS Site-to-Site VPN Documentation](https://docs.aws.amazon.com/vpn/latest/s2svpn/)
- [AWS Site-to-Site VPN Routing](https://docs.aws.amazon.com/vpn/latest/s2svpn/vpn-static-dynamic.html)
- [AWS Customer Gateway Firewall Requirements](https://docs.aws.amazon.com/vpn/latest/s2svpn/FirewallRules.html)
- [Amazon VPC Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)

---

**Author:** Hafiz Naeem Akmal

**GitHub:** [github.com/NaeemAkmal](https://github.com/NaeemAkmal)

**Project Repository:** [AWS Multi-Region Site-to-Site VPN Lab](https://github.com/NaeemAkmal/aws-multi-region-site-to-site-vpn-lab)

*Hands-on AWS Infrastructure, Networking and VPN Implementation | November 2026*
