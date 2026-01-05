# Set up Network Load Balancers

This document describes how to set up a **Google Cloud Network Load Balancer (NLB)** using Compute Engine instances and the `gcloud` CLI.

---

## Verify Authentication

### Check Active Account

```bash
gcloud auth list
```

**Output:**
```
ACTIVE: *
ACCOUNT: "ACCOUNT"
```

To set the active account, run:
```bash
gcloud config set account ACCOUNT
```

---

### Check Active Project

```bash
gcloud config list project
```

**Output:**
```
[core]
project = "PROJECT_ID"
```

---

## Task 1: Set the Default Region and Zone

Set the default region:
```bash
gcloud config set compute/region REGION
```

Set the default zone:
```bash
gcloud config set compute/zone ZONE
```

---

## Task 2: Create Multiple Web Server Instances

For this load balancing scenario, you will create three Compute Engine VM instances, install Apache on them, and configure a firewall rule to allow HTTP traffic.

Each instance uses the same network tag so they can be referenced together by firewall rules.  
Apache is installed automatically, and each instance serves a unique homepage.

---

### 1. Create VM Instance: www1

```bash
gcloud compute instances create www1 \
    --zone=Zone \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```

---

### 2. Create VM Instance: www2

```bash
gcloud compute instances create www2 \
    --zone=Zone \
    --tags=network-lb-tag \
    --machine-type=e2-small \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www2</h3>" | tee /var/www/html/index.html'
```

---

### 3. Create VM Instance: www3

Repeat the same steps as above to create a third VM named `www3`, updating the HTML content accordingly.

---

### 4. Create Firewall Rule to Allow HTTP Traffic

```bash
gcloud compute firewall-rules create www-firewall-network-lb \
    --target-tags network-lb-tag --allow tcp:80
```

---

### 5. List VM Instances

Run the following command to get the external IP addresses:

```bash
gcloud compute instances list
```

---

### 6. Verify Each Instance

Replace `IP_ADDRESS` with the external IP address of each VM:

```bash
curl http://IP_ADDRESS
```

---

## Task 3: Configure the Load Balancing Service

### 1. Create a Static External IP Address

```bash
gcloud compute addresses create network-lb-ip-1 \
  --region Region
```

**Output:**
```
Created [https://www.googleapis.com/compute/v1/projects/qwiklabs-gcp-03-xxxxxxxxxxx/regions/REGION/addresses/network-lb-ip-1].
```

---

### 2. Create a Legacy HTTP Health Check

```bash
gcloud compute http-health-checks create basic-check
```

---

## Task 4: Create the Target Pool and Forwarding Rule

A target pool is a group of backend instances that receive incoming traffic from a Network Load Balancer.  
All backend instances must reside in the same Google Cloud region.

---

### 1. Create Target Pool

```bash
gcloud compute target-pools create www-pool \
  --region Region --http-health-check basic-check
```

---

### 2. Add Instances to the Target Pool

```bash
gcloud compute target-pools add-instances www-pool \
    --instances www1,www2,www3
```

---

### 3. Create Forwarding Rule

```bash
gcloud compute forwarding-rules create www-rule \
    --region  Region \
    --ports 80 \
    --address network-lb-ip-1 \
    --target-pool www-pool
```

---

## Task 5: Send Traffic to Your Instances

Once the load balancer is configured, you can send traffic to the forwarding rule and observe traffic distribution across instances.

---

### 1. View Forwarding Rule Details

```bash
gcloud compute forwarding-rules describe www-rule --region REGION
```

---

### 2. Retrieve the External IP Address

```bash
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule   --region REGION   --format="json" | jq -r .IPAddress)
```

---

### 3. Display the IP Address

```bash
echo $IPADDRESS
```

---

### 4. Send Continuous Requests

```bash
while true; do curl -m1 $IPADDRESS; done
```
