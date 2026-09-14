# Secure AWS Infrastructure Deployment with Terraform
## Objective

This project documents the design, deployment, testing, and security of a small AWS network environment.

The current goal is to understand how traffic moves through a VPC, how public and private subnets differ, how routing and security controls affect connectivity, and how an EC2 instance communicates with the internet.

The environment is being built and tested manually first so that each AWS component is understood before the same infrastructure is codified with Terraform in the second stage of the project.

## Region

### AWS Region: `eu-west-2` — London

The London region was selected to keep project resources within a UK AWS region and align the environment with UK data-residency considerations.

## Current Architecture

The environment currently contains:

- VPC: `10.0.0.0/16`
- Public subnet: `10.0.1.0/24`
- Private subnet: `10.0.2.0/24`
- One Availability Zone
- Internet Gateway
- Separate public and private route tables
- One Ubuntu EC2 instance in the public subnet
- Security Group controlling access to the EC2 instance

The public subnet has a default route:

0.0.0.0/0 → Internet Gateway

The private subnet does not have a route to the Internet Gateway and currently contains no resources.

The EC2 instance has the private address:

10.0.1.6/24

and an AWS-assigned public IPv4 address used for internet connectivity.

## Internet Packet Path

Internet connectivity was tested successfully using:

sudo apt update

This demonstrated that the EC2 instance could reach Ubuntu package repositories through the public subnet.

## Outbound Path

When the EC2 instance initiates a connection to an internet destination, the traffic follows this path:
```
Application
    ↓
Linux network stack
    ↓
EC2 private IP: 10.0.1.6
    ↓
Elastic Network Interface (ENI)
    ↓
Security Group enforcement
    ↓
VPC routing decision
    ↓
Public subnet route table
    ↓
0.0.0.0/0 → Internet Gateway
    ↓
Internet Gateway
    ↓
Private IPv4 address translated to the instance's public IPv4 address
    ↓
Internet
    ↓
Destination server
```
The EC2 instance itself only has its private address configured on its network interface.

The public IPv4 address does not appear on the Linux interface.

The Internet Gateway handles the translation between the instance's private IPv4 address and its assigned public IPv4 address for internet traffic.

## Return Path

The response follows the reverse path:
```
Destination server
    ↓
Internet
    ↓
EC2 public IPv4 address
    ↓
Internet Gateway
    ↓
Public IPv4 address translated back to 10.0.1.6
    ↓
VPC routing
    ↓
Elastic Network Interface (ENI)
    ↓
Security Group state check
    ↓
Linux network stack
    ↓
Application
```
Security Groups are stateful, so return traffic for a connection initiated by the EC2 instance is automatically allowed.

## Planned Extensions

The following have not yet been implemented:

- Configure the EC2 instance to require IMDSv2
- Add an S3 bucket with:
    - Server-side encryption
    - Versioning
    - Block Public Access
- Enable AWS CloudTrail
- Introduce Terraform
- Store Terraform state locally during the first implementation
- Recreate the existing AWS infrastructure using Terraform
- Test `terraform plan`, `apply`, and `destroy`
- Add a private EC2 instance reachable only from the public instance
- Add GitHub Actions for Terraform validation
- Add Infrastructure as Code security scanning
- Improve Terraform state management after the local-state workflow is understood
