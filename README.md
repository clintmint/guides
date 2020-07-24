---
tags: guide
title: "Setting up a honeynet using MHN + GCP gcloud CLI"
---

Setting up a honeynet using MHN + GCP gcloud CLI
===

*Last tested July 2020*

> Notice: The following instructions assume you have already installed Google Cloud SDK from  https://cloud.google.com/sdk/docs/quickstarts

## Introduction

The Modern Honey Network (MHN) is used for the deployment of honeypots and collection of attack data using various sensors. This guide will use the Google Cloud Platform (GCP) Compute Engine for provisioning virtual machines (VM) for both the MHN administrative web application and its honeypot(s).

First gcloud is used to setup a new project and configure a firewall. The firewall for the MHN admin server is configured to allow in http, honeypot sensor data and geolocation data. The honeypot's firewall will allow in all TCP and UDP traffic from anywhere in the world.

Next the MHN Admin Web server is installed in a GCP instance, followed by a honeypot on a separate instance. After installation, MHN is configured through a web browser.

## Table of Contents

[TOC]

# Initialize Google SDK

> Notice for Windows Users: The commands throughout this guide are formatted to be run in a Linux (or macOS or Git Bash for Windows) terminal. In order to run them in the Google Cloud SDK Shell on Windows you will have to modify them first by using a search and replace function in a text editor.
> 
> Command pipelines using `grep` will need to be replaced with the equivalent Window's `findstr` command. Command blocks which span multiple lines will use a backslash (`\`) to indicate line continuation. These should be replaced with a caret symbol (`^`) if you are using the Windows Google Cloud SDK Shell.

```bash
gcloud init
gcloud projects create mhn-admin
gcloud config set project mhn-admin
```

## Enable billing

```bash
gcloud components update
gcloud components install beta
gcloud beta billing accounts list
gcloud beta billing projects link mhn-admin --billing-account 0X0X0X-0X0X0X-0X0X0X
```

## Enable Compute API

```bash
gcloud services list
gcloud services list --available
gcloud services list --available | grep compute
gcloud services enable compute
```

*Enabling services can take a minute or two, don't panic.*

## Find your region and zone

```bash
gcloud compute regions list
gcloud compute zones list
```

## Set region/zone

```bash
gcloud config set compute/zone us-central1-f
gcloud compute project-info add-metadata \
    --metadata google-compute-default-region=us-central1,google-compute-default-zone=us-central1-f
```

*Takes about 15 seconds to finish, should say "Updated ..."*

# Configure Firewall

```bash
gcloud compute firewall-rules list

gcloud compute firewall-rules create http \
    --allow tcp:80 \
    --description="Allow HTTP from Anywhere" \
    --direction ingress \
    --target-tags="mhn-admin"
    
gcloud compute firewall-rules create honeymap \
    --allow tcp:3000 \
    --description="Allow HoneyMap Feature from Anywhere" \
    --direction ingress \
    --target-tags="mhn-admin"

gcloud compute firewall-rules create hpfeeds \
    --allow tcp:10000 \
    --description="Allow HPFeeds from Anywhere" \
    --direction ingress \
    --target-tags="mhn-admin"

gcloud compute firewall-rules create wideopen \
    --description="Allow TCP and UDP from Anywhere" \
    --direction ingress \
    --priority=1000 \
    --network=default \
    --action=allow \
    --rules=tcp,udp \
    --source-ranges=0.0.0.0/0 \
    --target-tags="honeypot"
```

## Verify Firewall (Linux/MacOS bash)

```bash
gcloud compute firewall-rules list --format="table(
                name,
                network,
                direction,
                priority,
                sourceRanges.list():label=SRC_RANGES,
                allowed[].map().firewall_rule().list():label=ALLOW,
                targetTags.list():label=TARGET_TAGS,
                disabled
            )"
```

## Verify Firewall (Windows Google Cloud SDK Shell)

```cmd
gcloud compute firewall-rules list --format="table(name,network,direction,priority,sourceRanges.list():label=SRC_RANGES,allowed[].map().firewall_rule().list():label=ALLOW,targetTags.list():label=TARGET_TAGS,disabled)"
```

# MHN Admin

## Choose OS
*MHN is currently only supported on Ubuntu 18.04, 16.04 or CentOS 6.9*

```bash
gcloud compute images list | grep ubuntu-minimal
```
## Create VM

```bash
gcloud compute instances create "mhn-admin" \
    --machine-type "n1-standard-1" \
    --subnet "default" \
    --maintenance-policy "MIGRATE" \
    --tags "mhn-admin" \
    --image "ubuntu-minimal-1804-bionic-v20200703a" \
    --image-project "ubuntu-os-cloud" \
    --boot-disk-size "10" \
    --boot-disk-type "pd-standard" \
    --boot-disk-device-name "mhn-admin"
```

### Setup SSH

> Wait about 30 seconds before trying to SSH in before using 1 of the methods below.

#### Method 1

You can login using the following command. If you're on Windows, it'll try to login using PuTTY.

```bash
gcloud compute ssh mhn-admin
```

#### Method 2

```bash
gcloud compute config-ssh
```

*Open a new terminal window (PowerShell for Windows users) and use the SSH command from the previous command's output to log in.*

> NOTE: If you are using PowerShell to SSH in and you're getting "public key permission denied" errors it maybe due to a [bug](https://github.com/PowerShell/Win32-OpenSSH/issues/1197) in PowerShell which doesn't properly handle case sensitive usernames. Try specifying the username explicitly (i.e. `ssh UserName@host`) or use Git Bash instead.



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

# Honeypot

Return to the original terminal and enter the following to create the honeypot VM.
```bash
gcloud compute instances create "honeypot-1" \
    --machine-type "n1-standard-1" \
    --subnet "default" \
    --maintenance-policy "MIGRATE" \
    --tags "honeypot" \
    --image "ubuntu-minimal-1804-bionic-v20200703a" \
    --image-project "ubuntu-os-cloud" \
    --boot-disk-size "10" \
    --boot-disk-type "pd-standard" \
    --boot-disk-device-name "honeypot-1"
```
## Setup SSH

> Wait about 30 seconds before trying to SSH in before using 1 of the methods below.

#### Method 1

You can login using the following command. If you're on Windows, it'll try to login using PuTTY.

```bash
gcloud compute ssh honeypot-1
```

#### Method 2

```bash
gcloud compute config-ssh
```
*Open a New Terminal window (PowerShell/Git Bash for Windows users) and use the SSH command from the previous command's output to log in.*

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

#### Method 1

```bash
scp mhn-admin.us-central1-f.mhn-admin:~/session.json ~/session.json
```

#### Method 2

You can also use gcloud's built-in scp but it won't handle tildes (~) properly. This means you'll need to specify the absolute path to the source file. Replace USERNAME with your's. Note that usernames are case sensitive.

```bash
gcloud compute scp mhn-admin:/home/USERNAME/session.json ./session.json
```



# Clean Up

When you are done you can delete everything and stop billing by deleting your project.

```bash
gcloud projects list
gcloud projects delete mhn-admin
```