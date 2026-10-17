# Troubleshooting Log

This log records meaningful technical problems encountered while building the Secure AWS Infrastructure Deployment with Terraform project.

An issue belongs here if it takes more than about 10 minutes to resolve or teaches something important about AWS, Linux, networking, Terraform, Git, or security.

## Entry 001 - SSH Authentication Failed Because the Wrong Username Was Used

Date: 2026-09-13

### Problem

An SSH connection to the EC2 instance failed even though the instance was reachable and the correct private key file was being used.

The EC2 resource Name tag, `secure-aws-tf-web-01-dev`, was mistakenly used as the Linux login username.

### Evidence
```
PS C:\Users\VICTUS> ssh -i "$HOME\.ssh\aws-lab-eu-west-2.pem" secure-aws-tf-web-01-dev@[PUBLIC-IP-REDACTED]
The authenticity of host '[PUBLIC-IP-REDACTED] ([PUBLIC-IP-REDACTED])' can't be established.
ED25519 key fingerprint is SHA256:s/2gnY7o/rFClEYC3e60DHkjH5drNhE0hTFCX6rK154.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[PUBLIC-IP-REDACTED]' (ED25519) to the list of known hosts.
secure-aws-tf-web-01-dev@[PUBLIC-IP-REDACTED]: Permission denied (publickey).
```
### Hypothesis

My first assumption was that the SSH key might be incorrect or that the instance was rejecting the key.

### Investigation

- The SSH client reached the host successfully because the server returned its ED25519 host fingerprint.

- The connection did not time out, so basic network reachability to TCP port 22 was working.

- The error was `Permission denied (publickey)`, which indicated an authentication failure rather than a routing or connectivity failure.

- I checked the username being supplied to SSH and realised I had used the EC2 Name tag, `secure-aws-tf-web-01-dev`, as though it were the Linux account name.

- Because the instance was launched from an Ubuntu AMI, I retried the connection using the default Ubuntu username, `ubuntu`.

The corrected command was:
```
PS C:\Users\VICTUS> ssh -i "$HOME\.ssh\aws-lab-eu-west-2.pem" ubuntu@[PUBLIC-IP-REDACTED]
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.17.0-1017-aws x86_64)
ubuntu@ip-10-0-1-6:~$
```
### Root Cause

The EC2 resource Name tag was incorrectly treated as the operating system login identity. The Ubuntu AMI expected the default Linux user `ubuntu`, so SSH could not authenticate the non-existent `secure-aws-tf-web-01-dev` user.

### Fix

Retried the SSH connection using the Ubuntu AMI's default username:
```
ssh -i "$HOME\.ssh\aws-lab-eu-west-2.pem" ubuntu@[PUBLIC-IP-REDACTED]
```

The connection then succeeded.

### Lesson

The resource's display name and the Linux identity used for authentication are unrelated, and `Permission denied (publickey)` can mean the user does not exist rather than the key being wrong.

## Entry 002 — Public Route Removal Caused SSH to Time Out

Date: 2026-09-17

### Problem

I wanted to prove what the public subnet's `0.0.0.0/0 → Internet Gateway` route actually contributes to connectivity rather than simply assuming that attaching an Internet Gateway makes an instance public.

The test was:

- Establish a known-good SSH baseline.

- Remove the public subnet's default route to the Internet Gateway.

- Attempt a new SSH connection.

- Observe the effect on an already-established SSH session.

- Restore the route.

- Confirm SSH works again.

### Evidence

#### Capture 1 — Baseline before deleting the route

The baseline proves that the instance, key, username, SSH service, security group, and internet path were working immediately before the routing change.
```
PS C:\Users\VICTUS> ssh -v -i C:\Users\VICTUS\.ssh\aws-lab-eu-west-2.pem ubuntu@[PUBLIC-IP-REDACTED]
OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2
debug1: Connecting to [PUBLIC-IP-REDACTED] [[PUBLIC-IP-REDACTED]] port 22.
debug1: Connection established.
debug1: Remote protocol version 2.0, remote software version OpenSSH_9.6p1 Ubuntu-3ubuntu13.19
debug1: Authenticating to [PUBLIC-IP-REDACTED]:22 as 'ubuntu'
debug1: Offering public key: C:\\Users\\VICTUS\\.ssh\\aws-lab-eu-west-2.pem ED25519 SHA256:aOagTYjhjQH9XZr+gAphxZGjYIbm4Eqlq3z6Jrg+Bhk explicit
debug1: Server accepts key: C:\\Users\\VICTUS\\.ssh\\aws-lab-eu-west-2.pem ED25519 SHA256:aOagTYjhjQH9XZr+gAphxZGjYIbm4Eqlq3z6Jrg+Bhk explicit
Authenticated to [PUBLIC-IP-REDACTED] ([[PUBLIC-IP-REDACTED]]:22) using "publickey".
debug1: Entering interactive session.
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 7.0.0-1012-aws x86_64)
IPv4 address for ens5: 10.0.1.6
Last login: Wed Sep 16 22:57:36 2026 from [CLIENT-PUBLIC-IP-REDACTED]
```
#### Capture 2 — After deleting `0.0.0.0/0 → Internet Gateway`

The new SSH connection timed out after the public subnet's default route to the Internet Gateway was removed.
```
PS C:\Users\VICTUS> ssh -v -i C:\Users\VICTUS\.ssh\aws-lab-eu-west-2.pem ubuntu@[PUBLIC-IP-REDACTED]
OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2
debug1: Connecting to [PUBLIC-IP-REDACTED] [[PUBLIC-IP-REDACTED]] port 22.
debug1: connect to address [PUBLIC-IP-REDACTED] port 22: Connection timed out
ssh: connect to host [PUBLIC-IP-REDACTED] port 22: Connection timed out
PS C:\Users\VICTUS>
```
Observed: the SSH session that had already been established before the route was deleted also stopped working once the route was removed. No separate terminal capture was saved for that session, so this is recorded as an observation rather than reconstructed output.

#### Capture 3 — After restoring the route

Restoring `0.0.0.0/0 → Internet Gateway` returned the environment to the known-good state and a new SSH session succeeded again.
```
PS C:\Users\VICTUS> ssh -v -i C:\Users\VICTUS\.ssh\aws-lab-eu-west-2.pem ubuntu@[PUBLIC-IP-REDACTED]
OpenSSH_for_Windows_9.5p2, LibreSSL 3.8.2
debug1: Connecting to [PUBLIC-IP-REDACTED] [[PUBLIC-IP-REDACTED]] port 22.
debug1: Connection established.
debug1: Remote protocol version 2.0, remote software version OpenSSH_9.6p1 Ubuntu-3ubuntu13.19
debug1: Authenticating to [PUBLIC-IP-REDACTED]:22 as 'ubuntu'
debug1: Offering public key: C:\\Users\\VICTUS\\.ssh\\aws-lab-eu-west-2.pem ED25519 SHA256:aOagTYjhjQH9XZr+gAphxZGjYIbm4Eqlq3z6Jrg+Bhk explicit
debug1: Server accepts key: C:\\Users\\VICTUS\\.ssh\\aws-lab-eu-west-2.pem ED25519 SHA256:aOagTYjhjQH9XZr+gAphxZGjYIbm4Eqlq3z6Jrg+Bhk explicit
Authenticated to [PUBLIC-IP-REDACTED] ([[PUBLIC-IP-REDACTED]]:22) using "publickey".
debug1: Entering interactive session.
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 7.0.0-1012-aws x86_64)
IPv4 address for ens5: 10.0.1.6
Last login: Wed Sep 16 23:06:51 2026 from [CLIENT-PUBLIC-IP-REDACTED]
```
### Hypothesis

Prediction before the test: removing the public subnet's `0.0.0.0/0` route to the Internet Gateway would cause a new SSH connection from the internet to time out.

Result: confirmed.

My initial mechanism for that prediction was wrong. I first thought the route deletion would stop the inbound TCP SYN from reaching the instance.

### Investigation

- Confirmed SSH worked immediately before making the routing change.

- Removed only the public route `0.0.0.0/0 → Internet Gateway.`

- Left the instance, SSH key, username, security group, and SSH service unchanged.

- Retried SSH and received `Connection timed out`.

- Observed that the already-established SSH session also stopped working after the route was removed.

- Restored the same default route.

- Retried SSH and successfully authenticated again.

The controlled change isolated the subnet route as the variable responsible for the failure.

The original inbound-SYN explanation was then corrected. Traffic addressed to the instance's public IPv4 address is handled by the Internet Gateway, which translates the destination to the instance's private IPv4 address. The missing subnet default route prevents the instance from routing internet-bound response traffic back to the Internet Gateway.

For a new SSH connection, the client therefore sees a timeout because the TCP handshake cannot complete: even if the inbound SYN reaches the instance, the instance has no valid internet route for the SYN-ACK return path.

The same missing return path explains why the already-established SSH session also became unusable: an established TCP session still requires packets to travel successfully in both directions.

### Root Cause

Removing the public subnet's `0.0.0.0/0 → Internet Gateway` route removed the instance's route for internet-bound IPv4 traffic.

The EC2 instance still existed, still had its public IPv4 mapping, still had SSH running, and its security group had not changed, but it could no longer return traffic to the external SSH client through the Internet Gateway.

### Fix

Restored the public subnet route:

Destination: 0.0.0.0/0
Target:      Internet Gateway

A new SSH connection then succeeded immediately.

### Lesson

A public IPv4 address and attached Internet Gateway are not enough on their own; the subnet also needs a route to the Internet Gateway so internet connections have a working return path.

## Entry 003 — Terraform Plan Included More Resources Than Expected

Date: 2026-10-17

### Problem

After adding the EC2 Security Group and its ingress and egress rules, I expected Terraform to propose only the six newly added Security Group resources.

My prediction was:

```text
Plan: 6 to add, 0 to change, 0 to destroy.
```

Instead, Terraform reported:

```text
Plan: 11 to add, 0 to change, 0 to destroy.
```

The additional five resources were the public and private route-table resources from the previous stage.

### Evidence

The plan included:

```text
aws_route.public_internet
aws_route_table.private
aws_route_table.public
aws_route_table_association.private
aws_route_table_association.public
aws_security_group.ec2
aws_vpc_security_group_egress_rule.dns_tcp
aws_vpc_security_group_egress_rule.dns_udp
aws_vpc_security_group_egress_rule.http
aws_vpc_security_group_egress_rule.https
aws_vpc_security_group_ingress_rule.ssh
```

Terraform summarised the operation as:

```text
Plan: 11 to add, 0 to change, 0 to destroy.
```

### Hypothesis

My initial expectation was based only on the Terraform code I had most recently added.

I expected Terraform to plan the six Security Group-related resources because that was the current stage of the build.

### Investigation

I reviewed the resources listed in the plan rather than relying only on the final resource count.

The plan showed that five route-related resources were still waiting to be created:

```text
aws_route.public_internet
aws_route_table.private
aws_route_table.public
aws_route_table_association.private
aws_route_table_association.public
```

These resources had already been added to the Terraform configuration during the previous stage, but I had only run `terraform plan` for that stage and had not yet run `terraform apply`.

The six Security Group resources were therefore being planned together with the five previously configured but unapplied route resources:

```text
5 route resources
+
6 Security Group resources
=
11 resources to add
```

Terraform was not planning resources based on the most recent section of code I had edited.

It was comparing the complete desired configuration against the resources currently recorded in state.

### Root Cause

The route-table configuration existed in the `.tf` files but had not yet been applied.

Terraform therefore correctly detected that those five resources were still absent from the managed infrastructure.

When the Security Group configuration was subsequently added, the next `terraform plan` included both:

- the five previously unapplied route resources
- the six newly configured Security Group resources

### Fix

I reviewed the complete plan and confirmed that all eleven proposed resources were expected.

I then applied the configuration:

```powershell
terraform apply
```

Terraform created the outstanding route-table resources and the Security Group resources.

### Lesson

`terraform plan` does not operate only on the code that was most recently edited.

Terraform compares:

```text
Complete Terraform configuration
            ↓
       Terraform state
            ↓
   Real infrastructure
```

and calculates all actions required to make the infrastructure match the desired configuration.

A resource can therefore continue appearing in later plans if it has been written into the configuration but has not yet been applied.

This reinforced the importance of reading the actual resource addresses in a Terraform plan rather than judging the plan only by the final `X to add` count.


## Entry 004 — AWS CLI Was Installed but Not Available in the Existing PowerShell Session

Date: 2026-10-17

### Problem

Terraform needed authenticated access to AWS, so the AWS CLI was installed on the Windows workstation.

The AWS CLI installation completed successfully, but the existing PowerShell session initially could not use the `aws` command.

### Hypothesis

My first concern was that the AWS CLI installation might have failed or that the executable had been installed incorrectly.

### Investigation

The AWS CLI had been installed using:

```powershell
irm https://awscli.amazonaws.com/v2/install.ps1 | iex
```

The installation placed AWS CLI under:

```text
C:\Users\VICTUS\AppData\Local\Programs\Amazon\AWSCLIV2
```

The installation itself had therefore completed.

The issue was that the PowerShell process had already been running before the installer modified the Windows `PATH`.

The existing shell was still using the environment variables it inherited when it started and had not automatically picked up the newly added AWS CLI path.

A new terminal session was opened and the AWS CLI command then worked successfully.

### Root Cause

The AWS CLI was installed correctly, but the PowerShell session had not refreshed its environment after the installer updated `PATH`.

The issue was therefore related to the current Windows process environment rather than the AWS CLI installation itself.

### Fix

Closed the existing terminal and opened a new PowerShell session.

The AWS CLI was then available and its installation was verified before continuing with AWS authentication and Terraform.

### Lesson

On Windows, installing a command-line tool and updating the system or user `PATH` does not necessarily update environment variables inside terminal processes that are already running.

When a newly installed command is reported as unavailable, it is useful to distinguish between:

```text
application not installed
```

and:

```text
application installed
but current shell has stale PATH
```

before reinstalling or changing configuration unnecessarily.
