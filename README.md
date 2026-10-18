# Secure AWS Infrastructure Deployment with Terraform

## Objective

This project documents the design, deployment, testing, security, and Infrastructure as Code recreation of a small AWS network environment.

The project followed a **manual-first, Terraform-second** approach.

The environment was initially built manually so that the behaviour of each AWS component could be understood and tested independently. The same architecture was then recreated using Terraform and validated through a complete `destroy` → `apply` lifecycle.

The current Terraform-managed environment demonstrates:

- VPC design and subnet segmentation
- public and private routing
- Internet Gateway connectivity
- Security Group ingress and egress controls
- EC2 deployment
- IMDSv2 enforcement
- encrypted EC2 storage
- Terraform resource dependencies
- Terraform state
- infrastructure destruction and reproducible recreation
- post-deployment drift detection

---

## Region

### AWS Region: `eu-west-2` — London

The London region was selected to keep project resources within a UK AWS region and align the environment with UK data-residency considerations.

The current environment is deployed in:

```text
eu-west-2
```

with both subnets explicitly placed in:

```text
eu-west-2c
```

---

## Current Architecture

The current environment is managed with Terraform and contains:

- VPC: `10.0.0.0/16`
- Public subnet: `10.0.1.0/24`
- Private subnet: `10.0.2.0/24`
- Availability Zone: `eu-west-2c`
- Internet Gateway
- Separate public and private route tables
- Public default route to the Internet Gateway
- Explicit route-table associations for both subnets
- One Ubuntu EC2 instance in the public subnet
- AWS-assigned public IPv4 address for the EC2 instance
- Custom Security Group
- Restricted outbound Security Group rules
- SSH restricted to a trusted `/32`
- IMDSv2 required
- Metadata response hop limit set to `1`
- Encrypted `gp3` EC2 root volume
- Terraform outputs for key infrastructure identifiers
- Terraform local state for the current implementation

Terraform state currently contains `16` managed infrastructure resources.

The Ubuntu AMI lookup uses an `aws_ami` data source and is therefore not included in the managed-resource count.

---

## Network Design

```text
VPC: 10.0.0.0/16
│
├── Public Subnet: 10.0.1.0/24
│      │
│      ├── Ubuntu EC2
│      │
│      └── Public Route Table
│             │
│             └── 0.0.0.0/0 → Internet Gateway
│
└── Private Subnet: 10.0.2.0/24
       │
       └── Private Route Table
              │
              └── Local VPC route only
```

The private subnet currently contains no resources and has no route to the Internet Gateway.

---

## Public Subnet

The public subnet uses:

```text
10.0.1.0/24
```

Automatic public IPv4 assignment is disabled, the instance requests for IPV4 address via `associate_public_ip_address`

Its custom route table contains:

```text
10.0.0.0/16 → local
0.0.0.0/0   → Internet Gateway
```

The default route allows resources in the public subnet with public IPv4 addresses to communicate with the internet.

---

## Private Subnet

The private subnet uses:

```text
10.0.2.0/24
```

and does not automatically assign public IPv4 addresses.

Its custom route table contains only the VPC-local route:

```text
10.0.0.0/16 → local
```

There is currently:

```text
no 0.0.0.0/0 → Internet Gateway route
```

for the private subnet.

This intentionally keeps the subnet without direct internet connectivity.

---

## EC2 Instance

Terraform deploys one Ubuntu EC2 instance into the public subnet.

The instance configuration includes:

```text
Instance type: t3.micro
Operating system: Ubuntu 24.04 LTS
Public IPv4: AWS assigned
Root volume: gp3
Root-volume encryption: enabled
```

The Ubuntu AMI is discovered dynamically using an `aws_ami` data source rather than by permanently hard-coding a specific AMI ID.

The existing AWS key pair:

```text
aws-lab-eu-west-2
```

is referenced by Terraform but is not currently managed as a Terraform resource.

---

## Security Group

### Inbound

SSH is permitted only from one trusted public IPv4 address using a `/32` CIDR.

```text
TCP 22 → trusted-public-ip/32
```

No other inbound rules are configured.

The actual trusted public address is stored locally through Terraform input configuration and is not committed to the repository.

### Outbound

The default AWS rule:

```text
All traffic → 0.0.0.0/0
```

was removed.

Terraform now manages the following outbound rules:

```text
TCP 80  → 0.0.0.0/0
TCP 443 → 0.0.0.0/0
UDP 53  → 10.0.0.2/32
TCP 53  → 10.0.0.2/32
```

HTTP and HTTPS remain broadly permitted because package-management traffic reaches multiple external destinations whose addresses may change.

Testing showed that `apt update` contacted at least:

```text
eu-west-2.ec2.archive.ubuntu.com
security.ubuntu.com
```

Restricting web egress to a single static destination would therefore be unreliable without a more advanced egress-control mechanism.

### Route 53 Resolver Limitation

The intended DNS resolver is:

```text
10.0.0.2
```

which is the Amazon-provided Route 53 Resolver for the VPC.

The DNS rules document the intended resolver and no general outbound port `53` rule exists for arbitrary external resolvers.

However, AWS Security Groups do not filter traffic to or from the Amazon-provided Route 53 Resolver.

This is a known limitation of the current control.

Further DNS policy enforcement would require a DNS-aware service such as Route 53 Resolver DNS Firewall.

Full egress testing is recorded in [`docs/evidence.md`](docs/evidence.md).

---

## EC2 Metadata Security

The EC2 instance requires **IMDSv2**.

Terraform configures:

```hcl
metadata_options {
  http_endpoint               = "enabled"
  http_tokens                 = "required"
  http_put_response_hop_limit = 1
}
```

Testing confirmed that a metadata request without a token returned:

```text
HTTP/1.1 401 Unauthorized
```

while the IMDSv2 token workflow successfully returned instance metadata.

The metadata response hop limit is:

```text
1
```

which limits metadata responses to a single network hop.

IMDSv2 enforcement was validated both before and after the Terraform destroy-and-rebuild test.

Full evidence is recorded in [`docs/evidence.md`](docs/evidence.md).

---

## Internet Packet Path

Internet connectivity from the EC2 instance was tested successfully using:

```bash
sudo apt update
```

The instance successfully reached Ubuntu package repositories.

### Outbound Path

```text
Application
    ↓
Linux network stack
    ↓
EC2 private IPv4 address
    ↓
Elastic Network Interface
    ↓
Security Group
    ↓
VPC routing
    ↓
Public subnet route table
    ↓
0.0.0.0/0 → Internet Gateway
    ↓
Internet Gateway
    ↓
Public IPv4 translation
    ↓
Internet
    ↓
Destination
```

The public IPv4 address does not appear directly on the EC2 Linux network interface.

For IPv4 internet communication, the Internet Gateway handles the mapping between the instance's private and public addresses.

---

## Return Path

Return traffic follows the reverse path:

```text
Destination
    ↓
Internet
    ↓
EC2 public IPv4 address
    ↓
Internet Gateway
    ↓
Private IPv4 address
    ↓
VPC routing
    ↓
Elastic Network Interface
    ↓
Security Group state check
    ↓
Linux network stack
    ↓
Application
```

Security Groups are stateful.

Return traffic belonging to an allowed connection is automatically permitted without requiring a separate reverse-direction Security Group rule.

---

## Route-Table Validation Test

The importance of the public subnet's default route was tested during the manual stage by temporarily removing:

```text
0.0.0.0/0 → Internet Gateway
```

After the route was removed:

- new SSH connections timed out
- an already-established SSH session became unusable
- restoring the route restored connectivity

The initial prediction that SSH would fail was correct.

However, the original explanation was corrected during the investigation.

The failure was not simply caused by the inbound SYN being unable to reach the instance. The missing route prevented the instance from sending internet-bound return traffic through the Internet Gateway, preventing the TCP handshake from completing.

The test demonstrated that:

```text
Public IPv4
+
Internet Gateway
+
Security Group allowing SSH
```

are not sufficient without:

```text
0.0.0.0/0 → Internet Gateway
```

in the subnet's route table.

The complete troubleshooting process is recorded in [`docs/troubleshooting-log.md`](docs/troubleshooting-log.md).

---

## Terraform Infrastructure as Code

The manually tested AWS network architecture has now been recreated using Terraform.

Terraform manages:

- VPC
- public subnet
- private subnet
- Internet Gateway
- public route table
- private route table
- public default route
- route-table associations
- custom Security Group
- SSH ingress rule
- HTTP egress rule
- HTTPS egress rule
- DNS TCP egress rule
- DNS UDP egress rule
- public EC2 instance

Terraform resource references are used instead of hard-coded AWS resource IDs.

For example:

```hcl
vpc_id = aws_vpc.main.id
```

and:

```hcl
subnet_id = aws_subnet.public.id
```

allow Terraform to construct the dependency graph automatically.

---

## Terraform Project Structure

```text
terraform/
├── main.tf
├── providers.tf
├── versions.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars          # local / gitignored
├── terraform.tfstate         # local / gitignored
├── .terraform.lock.hcl       # committed
└── .terraform/               # local / gitignored
```

`terraform.tfvars` contains environment-specific values such as the trusted SSH source CIDR and is excluded from Git.

Terraform state is currently stored locally in `terraform.tfstate` and is also excluded from Git.

The provider lock file:

```text
.terraform.lock.hcl
```

is committed so that Terraform provider selections remain consistent.

---

## Terraform Lifecycle Validation

The environment was tested through the full Terraform lifecycle:

```text
terraform init
    ↓
terraform fmt
    ↓
terraform validate
    ↓
terraform plan
    ↓
terraform apply
```

After the first deployment, SSH, internet connectivity and IMDSv2 were successfully validated.

A complete destruction and recreation test was then performed.

### Destroy

Before destruction:

```text
terraform plan -destroy
```

reported:

```text
Plan: 0 to add, 0 to change, 16 to destroy.
```

The environment was then removed using:

```bash
terraform destroy
```

Terraform reported:

```text
Destroy complete! Resources: 16 destroyed.
```

Afterwards:

```bash
terraform state list
```

returned no managed resources.

### Rebuild

The Terraform configuration remained unchanged.

Running:

```bash
terraform plan
```

after destruction produced:

```text
Plan: 16 to add, 0 to change, 0 to destroy.
```

Terraform then recreated the complete environment:

```bash
terraform apply
```

with:

```text
Apply complete! Resources: 16 added, 0 changed, 0 destroyed.
```

No VPC, subnet, route table, Internet Gateway, Security Group, or EC2 resource was manually recreated between the destroy and apply operations.

---

## Post-Rebuild Validation

The rebuilt environment was validated independently.

### SSH

SSH connectivity to the newly created EC2 instance succeeded.

```bash
ssh -i "$HOME\.ssh\aws-lab-eu-west-2.pem" ubuntu@[PUBLIC-IP-REDACTED]
```

### Internet Connectivity

The rebuilt instance successfully executed:

```bash
sudo apt update
```

and reached both the regional Ubuntu package mirror and `security.ubuntu.com`.

### IMDSv2

A tokenless metadata request returned:

```text
401 Unauthorized
```

while the IMDSv2 token workflow successfully returned the new instance ID.

This confirmed that the IMDSv2 security configuration survived the complete destroy-and-rebuild lifecycle.

### Final Drift Check

After validation:

```bash
terraform plan
```

returned:

```text
No changes. Your infrastructure matches the configuration.
```

This confirmed that the deployed AWS infrastructure and Terraform configuration were aligned after recreation.

Full Terraform lifecycle evidence is recorded in [`docs/evidence.md`](docs/evidence.md).

---

## Terraform Outputs

The project exposes useful Terraform outputs including:

```text
vpc_id
public_subnet_id
private_subnet_id
ec2_instance_id
ec2_private_ip
ec2_public_ip
```

These allow infrastructure values to be retrieved without manually looking them up in the AWS console.

For example:

```bash
terraform output -raw ec2_public_ip
```

returns the current public IPv4 address of the Terraform-managed EC2 instance.

---

## Earlier Manual Security Testing

Before the Terraform implementation, additional AWS controls were tested manually and documented as part of the learning stage.

These included:

- S3 server-side encryption
- S3 versioning and non-current-version recovery
- S3 Block Public Access behaviour
- anonymous S3 access testing
- CloudTrail management-event logging
- CloudTrail change-and-rollback reconstruction

The CloudTrail trail and its destination S3 bucket used during that test have since been deleted and are **not part of the current live architecture**.

The historical evidence remains documented in [`docs/evidence.md`](docs/evidence.md).

---

## Planned Extensions

The following have not yet been implemented:

- add a private EC2 instance reachable only through controlled internal access
- validate communication between the public and private tiers
- evaluate replacing SSH administration with AWS Systems Manager Session Manager
- manage or replace the currently external EC2 key pair through a more reproducible access strategy
- migrate local Terraform state to a remote backend with state locking after the local-state workflow is fully understood
- evaluate more advanced HTTP/HTTPS egress filtering
- refactor repeated Terraform configuration into reusable modules after the raw-resource implementation is fully understood

---

## Evidence

Detailed technical evidence is maintained separately:

- [`docs/evidence.md`](docs/evidence.md) — implementation and validation evidence
- [`docs/troubleshooting-log.md`](docs/troubleshooting-log.md) — failures, hypotheses, root causes, corrections, and lessons learned

Sensitive identifiers such as public IP addresses, AWS account IDs, resource identifiers, credentials, and trusted source addresses are redacted from public evidence where they are not required to demonstrate the technical behaviour.
