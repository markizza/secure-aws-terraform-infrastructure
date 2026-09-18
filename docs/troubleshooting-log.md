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
