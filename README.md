---
tags: guide
title: "Setting up a honeynet using MHN + AWS CLI"
---

Setting up a honeynet using MHN + AWS CLI
===

*Last tested July 2020*

## Introduction

The [Modern Honey Network](https://github.com/pwnlandia/mhn) (MHN) is used for the deployment of honeypots and collection of attack data using various sensors. This guide will use the Amazon Web Services (AWS) EC2 for provisioning virtual machines (VM) for both the MHN administrative web application and its honeypot(s).

First aws is used to setup a new project and configure a firewall. The firewall for the MHN admin server is configured to allow in http, honeypot sensor data and geolocation data. The honeypot's firewall will allow in all TCP and UDP traffic from anywhere in the world.

Next the MHN Admin Web server is installed in a AWS instance, followed by a honeypot on a separate instance. After installation, MHN is configured through a web browser.

## Table of Contents

[TOC]

## Get a free AWS account

and $300 worth of credit 

https://aws.amazon.com/free/

## Configure AWS CLI

Install AWS CLI from https://aws.amazon.com/cli/

Sign in to the link below to create your access keys. You can only show it and download once. Keep it safe as this is the root user's key. 

> Note that in a production environment you'd disable the root account and create a user group for admins. Since this is only for a homework assignment we will accept the risk and just use the root key.

https://console.aws.amazon.com/iam/

Decide which region you want to use.

https://docs.aws.amazon.com/general/latest/gr/rande.html

Now configure aws with your access key, secret, region and output type.

```
aws configure
```

Confirm your settings

```
aws configure list
```

Change output type to whatever you'd like (json,yaml,text,table). Table looks pretty on a big terminal.

```
aws configure set output table
```

You can override output for a single command

```
aws configure list --output json
```

Getting help

```
aws ec2 help
```

## Create SSH keys

### Linux/macOS

```
aws ec2 create-key-pair --key-name amznKey --query 'KeyMaterial' --output text > amznKey.pem
chmod 400 amznKey.pem
```

### Powershell

Need to pipe it to ASCII text otherwise powershell defaults to UTF encoding.

```
aws ec2 create-key-pair --key-name amznKey --query 'KeyMaterial' --output text | out-file -encoding ascii -filepath amznKey.pem
```

## Security Groups

Security Groups are the firewalls in AWS. We'll create 1 for mhn-admin and 1 for honeypots. Make note of the 1 vpc-* ID and the 2 sg-* IDs for each firewall.

Get the VPC ID

```
aws ec2 describe-vpcs --output json | findstr VpcId
aws ec2 describe-vpcs --output json | grep VpcId
```

### MHN Admin Firewall

Create security group for mhn-admin

```
aws ec2 create-security-group --group-name mhn-admin --description "MHN-Admin Firewall" --vpc-id vpc-1234567890
```

Create rules to allow SSH and HTTP from your ip or subnet. Replace sg-* with output from previous command. Replace last IP address with your own IP or subnet. If it's a single host IP use the /32 suffix.

```
aws ec2 authorize-security-group-ingress --group-id sg-0123456789 --protocol tcp --port 22 --cidr 192.168.0.1/24
aws ec2 authorize-security-group-ingress --group-id sg-0123456789 --protocol tcp --port 80 --cidr 192.168.0.1/24
```

Create rules to allow hpfeeds and honeymap traffic. Replace sg-* with the security group for MHN-Admin. Leave IP as 0.0.0.0/0 to indicate all IPs.

```
aws ec2 describe-vpcs --output json | findstr CidrBlock
aws ec2 authorize-security-group-ingress --group-id sg-0123456789 --protocol tcp --port 3000 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-0123456789 --protocol tcp --port 10000 --cidr 0.0.0.0/0
```

Verify rules

```
aws ec2 describe-security-groups --group-ids sg-0123456789
```

### Honeypot Firewall

Create security group for honeypots. Replace vpc-* 

```
aws ec2 create-security-group --group-name mhn-admin --description "Honeypot Firewall" --vpc-id vpc-1234567890
```

Create rules to allow all UDP and TCP traffic from everywhere. Replace sg-* with output from previous command. 

```
aws ec2 authorize-security-group-ingress --group-id sg-abcdefghi --protocol tcp --port 0-65535 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-abcdefghi --protocol udp --port 0-65535 --cidr 0.0.0.0/0
```

Verify rules

```
aws ec2 describe-security-groups --group-ids sg-abcdefghi
```

# Create MHN-Admin VM

Find latest AWS image-id for Ubuntu 18.04 minimal in your region. Check CreationDates. 

```
aws ec2 describe-images --filters "Name=product-code,Values=dc3kgx9eyqg3we2d6fa4ndr8"
```

Get the latest version by checking the creation dates

![](https://i.imgur.com/q8WIbRT.png)

Create the mhn-admin instance. Replace sg-* with mhn's security group.

```
aws ec2 run-instances --image-id ami-0ae68c2e193fe154b --count 1 --instance-type t2.micro --key-name amznKey --security-group-ids sg-0123456789
```

Get the dns name after a minute or 2. Replace i-* with output from last command.

```
aws ec2 describe-instances --instance-ids i-1234567890
```

SSH in. Default username is ubuntu.

```
ssh -i .\amznKey.pem ubuntu@ec2-123-456-789-120.compute-1.amazonaws.com
```

### Update, Download and Install MHN-Admin

```bash
sudo apt update && sudo apt install git python-magic -y

cd /opt/
sudo git clone https://github.com/pwnlandia/mhn.git
cd mhn/

sudo ./install.sh
```

### Configure

Take note of the 'Superuser email', 'Superuser password' and 'Server base url'. These will be be your credentials for logging in to the MHN Admin webserver.

```bash
===========================================================
MHN Configuration
===========================================================
Do you wish to run in Debug mode?: y/n n
Superuser email: someone@something.somewhere
Superuser password: passwordHidden
Server base url ["http://1.2.3.4"]:
Honeymap url ["http://1.2.3.4:3000"]:
Mail server address ["localhost"]: 
Mail server port [25]: 
Use TLS for email?: y/n n
Use SSL for email?: y/n n
Mail server username [""]: 
Mail server password [""]: 
Mail default sender [""]: 
Path for log file ["mhn.log"]:
```

More config questions about 15 minutes later...
```bash
Would you like to integrate with Splunk? (y/n) n
Would you like to install ELK? (y/n) n
```
Last one...
```bash
Would you like to add MHN rules to UFW? (y/n) n
```

### Confirm successful installation

```bash
sudo /etc/init.d/nginx status
sudo /etc/init.d/supervisor status
sudo supervisorctl status
```

# Create Honeypot VM

Create the honeypot instance with same ubuntu image-id. Replace sg-* with honeypot's security group.

```
aws ec2 run-instances --image-id ami-0ae68c2e193fe154b --count 1 --instance-type t2.micro --key-name amznKey --security-group-ids sg-0123456789
```

Get the dns name after a minute or 2. Replace i-* with output from previous command.

```
aws ec2 describe-instances --instance-ids i-1234567890
```

SSH in. Default username is ubuntu. Replace ec2-*.compute* with output from previous command.

```
ssh -i .\amznKey.pem ubuntu@ec2-123-456-789-120.compute-1.amazonaws.com
```


# Deploy Honeypot Sensors

If you forgot to write down the public IP of the MHN admin site you can use the following command in the MHN Admin's SSH terminal session:

`curl ipinfo.io/ip`

**Connect to the MHN Admin IP address in your favorite web browser and login.**

![](https://i.imgur.com/opgqiIb.png)

**Click Deploy and Select Script.**

![](https://i.imgur.com/NGRSa8w.png)

**Copy Deploy Command.**

![](https://i.imgur.com/1tV80WZ.png)

**Enter the command in the *honeypot terminal* and confirm successful deployment.**

![](https://i.imgur.com/qhVC2WF.png)

# Dump Sensor Database

After you are finished gathering honey you can dump the sensor database from an mhn-admin terminal.

```bash
mongoexport --db mnemosyne --collection session > session.json
```

The file may be quite large if the honeynet has been running for several days. You can shrink it down to 5 mebibytes with the following command.

```bash
truncate --size="<5M" session.json
```

# Transfer sensor data back to your computer

From the host machine, copy it back using SCP

```bash
scp -i .\amznKey.pem ubuntu@ec2-123-456-789-120.compute-1.amazonaws.com:~/session.json ~/session.json
```

# Clean Up

When you are done you can delete everything and stop billing by deleting your ec2 instances.

```bash
aws ec2 describe-instances | findstr InstanceId
aws ec2 terminate-instances --instance-ids i-023c8e21587f623bf i-0a800d9026c3e8bae
```