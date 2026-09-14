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

Lesson

The resource's display name and the Linux identity used for authentication are unrelated, and `Permission denied (publickey)` can mean the user does not exist rather than the key being wrong.
