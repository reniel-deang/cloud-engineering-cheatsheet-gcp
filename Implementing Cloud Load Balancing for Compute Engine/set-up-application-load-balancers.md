# Set Up Application Load Balancers

This document describes how to set up a **Google Cloud Application Load Balancer** using Compute Engine instances and the `gcloud` CLI.

---

## Verify Authentication and Project

```bash
gcloud auth list
```

```bash
gcloud config list project
```

---

## Task 1: Set the Default Region and Zone

Set the default region:
```bash
gcloud config set compute/region Region
```

Set the default zone:
```bash
gcloud config set compute/zone Zone
```

---

## Task 2: Create Multiple Web Server Instances

For this load balancing scenario, you will create three Compute Engine VM instances and install Apache on them.  
A firewall rule will be added to allow HTTP traffic to reach the instances.

The zone is set to `Zone`.  
The `tags` field allows referencing all instances at once (for example, in firewall rules).  
Each instance installs Apache and serves a unique home page.

---

### 1. Create VM Instance: www1

```bash
gcloud compute instances create www1   --zone=Zone   --tags=network-lb-tag   --machine-type=e2-small   --image-family=debian-11   --image-project=debian-cloud   --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    service apache2 restart
    echo "<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```

---

### 2. Create VM Instances: www2 and www3

Repeat the same command above for `www2` and `www3`, updating the HTML content accordingly.

---

### 3. Create Firewall Rule to Allow HTTP Traffic

```bash
gcloud compute firewall-rules create www-firewall-network-lb   --target-tags network-lb-tag   --allow tcp:80
```

---

### 4. List VM Instances

```bash
gcloud compute instances list
```

---

### 5. Verify Each Instance

Replace `IP_ADDRESS` with the external IP address of each VM:

```bash
curl http://[IP_ADDRESS]
```

---

## Task 3: Create an Application Load Balancer

Application Load Balancing is implemented on **Google Front Ends (GFE)**, which are globally distributed and operate on Google’s global network.

Requests are routed to the closest instance group that has sufficient capacity.  
If the closest group does not have enough capacity, traffic is routed to the next closest healthy group.

To use an Application Load Balancer, backend VMs must be part of an **instance group**.  
For this setup, managed instance groups (MIGs) are used.

---

### 1. Create an Instance Template

```bash
gcloud compute instance-templates create lb-backend-template   --region=Region   --network=default   --subnet=default   --tags=allow-health-check   --machine-type=e2-medium   --image-family=debian-11   --image-project=debian-cloud   --metadata=startup-script='#!/bin/bash
    apt-get update
    apt-get install apache2 -y
    a2ensite default-ssl
    a2enmod ssl
    vm_hostname="$(curl -H "Metadata-Flavor:Google"     http://169.254.169.254/computeMetadata/v1/instance/name)"
    echo "Page served from: $vm_hostname" | tee /var/www/html/index.html
    systemctl restart apache2'
```

---

### 2. Create a Managed Instance Group

```bash
gcloud compute instance-groups managed create lb-backend-group   --template=lb-backend-template   --size=2   --zone=Zone
```

---

### 3. Create Firewall Rule for Health Checks

```bash
gcloud compute firewall-rules create fw-allow-health-check   --network=default   --action=allow   --direction=ingress   --source-ranges=130.211.0.0/22,35.191.0.0/16   --target-tags=allow-health-check   --rules=tcp:80
```

Note: This ingress rule allows traffic from Google Cloud health checking systems.

---

### 4. Create a Global Static External IP Address

```bash
gcloud compute addresses create lb-ipv4-1   --ip-version=IPV4   --global
```

View the reserved IP address:

```bash
gcloud compute addresses describe lb-ipv4-1   --format="get(address)"   --global
```

---

### 5. Create a Health Check

```bash
gcloud compute health-checks create http http-basic-check   --port 80
```

---

### 6. Create a Backend Service

```bash
gcloud compute backend-services create web-backend-service   --protocol=HTTP   --port-name=http   --health-checks=http-basic-check   --global
```

---

### 7. Add Instance Group to Backend Service

```bash
gcloud compute backend-services add-backend web-backend-service   --instance-group=lb-backend-group   --instance-group-zone=Zone   --global
```

---

### 8. Create a URL Map

```bash
gcloud compute url-maps create web-map-http   --default-service web-backend-service
```

---

### 9. Create a Target HTTP Proxy

```bash
gcloud compute target-http-proxies create http-lb-proxy   --url-map web-map-http
```

---

### 10. Create a Global Forwarding Rule

```bash
gcloud compute forwarding-rules create http-content-rule   --address=lb-ipv4-1   --global   --target-http-proxy=http-lb-proxy   --ports=80
```

---

## Task 4: Test Traffic Sent to Your Instances

1. In the Google Cloud Console, search for **Load balancing**.
2. Select the load balancer named `web-map-http`.
3. In the **Backend** section, confirm that the VMs are **Healthy**.
4. Open a web browser and go to:

```
http://IP_ADDRESS/
```

Replace `IP_ADDRESS` with the load balancer’s external IP address.
