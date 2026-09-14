# Project Evidence
## 1. Successful SSH Connection

This proves that the EC2 instance is reachable over SSH using the correct Ubuntu username and private key.
```
PS C:\Users\VICTUS> ssh -i "$HOME\.ssh\aws-lab-eu-west-2.pem" ubuntu@[PUBLIC-IP-REDACTED]
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.17.0-1017-aws x86_64)
ubuntu@ip-10-0-1-6:~$
```
## 2. SSH Authentication Failure with the Wrong Username

This proves that TCP connectivity reached the host and the SSH service responded, but authentication failed because the EC2 Name tag was used as the login username instead of the Ubuntu AMI's default user.
```
PS C:\Users\VICTUS> ssh -i "$HOME\.ssh\aws-lab-eu-west-2.pem" secure-aws-tf-web-01-dev@[PUBLIC-IP-REDACTED]
The authenticity of host '[PUBLIC-IP-REDACTED] ([PUBLIC-IP-REDACTED])' can't be established.
ED25519 key fingerprint is SHA256:s/2gnY7o/rFClEYC3e60DHkjH5drNhE0hTFCX6rK154.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[PUBLIC-IP-REDACTED]' (ED25519) to the list of known hosts.
secure-aws-tf-web-01-dev@[PUBLIC-IP-REDACTED]: Permission denied (publickey).
```
## 3. Internet Connectivity from the Public EC2 Instance

This proves that the instance can reach Ubuntu package repositories through the public subnet's internet route.
```
ubuntu@ip-10-0-1-6:~$ sudo apt update
Hit:1 http://eu-west-2.ec2.archive.ubuntu.com/ubuntu noble InRelease
Get:2 http://eu-west-2.ec2.archive.ubuntu.com/ubuntu noble-updates InRelease [126 kB]
Get:3 http://eu-west-2.ec2.archive.ubuntu.com/ubuntu noble-backports InRelease [126 kB]
[... output trimmed ...]
Fetched 36.9 MB in 6s (6247 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
143 packages can be upgraded. Run 'apt list --upgradable' to see them.
```
## 4. Private Network Interface and Public-IP NAT Behaviour

This proves that the EC2 instance holds only its private 10.0.1.6/24 address on ens5; the public IPv4 address is not configured on the instance itself because the Internet Gateway performs one-to-one NAT between the public address and the instance's private address.
```
ubuntu@ip-10-0-1-6:~$ ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever

2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 06:f1:73:6d:d6:4d brd ff:ff:ff:ff:ff:ff
    altname enp0s5
    inet 10.0.1.6/24 metric 100 brd 10.0.1.255 scope global dynamic ens5
       valid_lft 2443sec preferred_lft 2443sec
    inet6 fe80::4f1:73ff:fe6d:d64d/64 scope link
       valid_lft forever preferred_lft forever
```
