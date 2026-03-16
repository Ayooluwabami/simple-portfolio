# Lightsail + EC2 Hybrid Architecture Lab

> **A cost-optimised two-tier AWS setup where Lightsail handles all public-facing traffic and EC2 runs as a fully private compute layer — connected via VPC peering.**

This lab was inspired by a comment from [Muhammad Harith Omar](https://www.linkedin.com/in/muhammadharithomar/) on LinkedIn:

> *"Lightsail is cheaper for the public-facing interface if you need many GBs of bandwidth per month. You don't get much CPU but enough SSD for cache. Then connect to your EC2."*

---

## Architecture Overview

```
Internet → Lightsail (Nginx reverse proxy) → VPC Peering → EC2 (Apache, private) → response
```

| Layer | Service | Role |
|---|---|---|
| Public edge | AWS Lightsail | Nginx reverse proxy, public IP, bundled bandwidth |
| Private compute | AWS EC2 | Apache web server, no public HTTP access |
| Network link | VPC Peering | Private routing between Lightsail and EC2 |
| Access control | Security Groups | EC2 port 80 restricted to Lightsail CIDR only |

### Why This Pattern

AWS EC2 charges ~$0.09/GB for outbound data beyond the free tier. Lightsail's $5/month plan includes **1 TB of outbound transfer** at a flat rate. By placing Lightsail at the public-facing layer, you absorb all user-facing bandwidth at a predictable, lower cost — while keeping compute on EC2 where it belongs.

---

## Prerequisites

- An AWS account with access to both EC2 and Lightsail consoles
- Basic familiarity with AWS VPC, EC2, and SSH
- Completion of [Lab 1 — EC2 Apache Web Server in a Custom VPC](../lab-01-ec2-apache-custom-vpc) is recommended but not required
- An SSH key pair created in eu-west-2 (London)

---

## Objectives

By the end of this lab you will have:

- Provisioned an EC2 instance running Apache inside a custom VPC
- Created a Lightsail instance running Nginx as a reverse proxy
- Established VPC peering between Lightsail and the default VPC
- Locked down EC2 so it only accepts HTTP traffic from Lightsail's private CIDR
- Validated end-to-end traffic flow from browser through to Apache

---

## Estimated Time

**90–120 minutes**

---

## Cost Estimate

| Resource | Cost |
|---|---|
| Lightsail $5/month plan | $5.00/mo |
| EC2 t3.micro (on-demand) | ~$8.32/mo |
| VPC peering data transfer | < $1.00/mo |
| **Total** | **~$13–$16/mo** |

> **Remember to stop or terminate your EC2 instance and delete your Lightsail instance after completing the lab to avoid ongoing charges.**

---

## Lab Tasks

### Task 1 — Launch EC2 and Install Apache

**1.1 — Launch the EC2 instance**

1. Open the [EC2 console](https://console.aws.amazon.com/ec2) and click **Launch Instance**
2. Configure the instance:
   - **Name:** `cba_instance`
   - **AMI:** Amazon Linux 2023
   - **Instance type:** t3.micro
   - **Key pair:** Select or create a key pair
   - **VPC:** Select your custom VPC (`cba-vpc`, CIDR `10.0.0.0/16`) or the default VPC
   - **Subnet:** Select your public subnet
   - **Auto-assign public IP:** Enable
3. Under **Security Group**, create a new group named `cba_security_group` with the following inbound rules:

   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | SSH | TCP | 22 | 0.0.0.0/0 |
   | HTTP | TCP | 80 | 0.0.0.0/0 |
   | HTTPS | TCP | 443 | 0.0.0.0/0 |

4. Click **Launch Instance**

**1.2 — Install Apache**

Connect to the instance via EC2 Instance Connect or SSH, then run:

```bash
sudo su
yum update -y
yum install httpd -y
systemctl start httpd
systemctl enable httpd
```

Verify Apache is running:

```bash
systemctl status httpd
```

You should see `Active: active (running)`. Load the EC2 public IP in a browser — the default Apache test page should appear.

---

### Task 2 — Deploy a Custom Index Page

Replace the default Apache page with your own:

```bash
cd /var/www/html
sudo nano index.html
```

Paste your custom HTML content and save. The page will be served immediately — no Apache restart required.

> This lab uses a portfolio-style index page. You can find the example used in this project at [`index.html`](./index.html).

---

### Task 3 — Create the Lightsail Instance

1. Open the [Lightsail console](https://lightsail.aws.amazon.com) — note this is a separate service from EC2
2. Click **Create instance**
3. Configure:
   - **Region:** eu-west-2 (London) — must match your EC2 region
   - **Platform:** Linux/Unix
   - **Blueprint:** OS Only → Amazon Linux 2
   - **Plan:** $5/month (1 vCPU, 512 MB RAM, 20 GB SSD, 1 TB transfer)
   - **Instance name:** `lightsail-proxy`
4. Click **Create instance**

Once running, note the **public IP** and **private IP** of the Lightsail instance. The private IP will be in the `172.26.x.x` range, confirming it lives in the Lightsail-managed VPC (`172.26.0.0/16`).

---

### Task 4 — Enable VPC Peering

> ⚠️ **Important:** Lightsail's built-in VPC peering only connects to the **AWS default VPC**, not a custom VPC. If your EC2 instance is in a custom VPC, see the note below.

1. In the Lightsail console, go to **Account → Advanced**
2. Under **VPC peering**, click **Enable VPC peering** for your region (eu-west-2)
3. Lightsail will automatically create a peering connection to the default VPC

**If your EC2 is in a custom VPC:**

Lightsail peering with custom VPCs is not supported through the console. The Lightsail VPC belongs to an AWS-managed account and cross-account peering cannot be accepted through the Lightsail interface. The solution is to launch your EC2 instance in the **default VPC** instead.

To find the Lightsail VPC ID (for reference), run this from inside the Lightsail terminal:

```bash
MAC=$(curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/)
curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/${MAC}vpc-id
```

---

### Task 5 — Move EC2 to the Default VPC (if needed)

If your original EC2 was in a custom VPC, launch a new instance in the default VPC:

1. Launch a new EC2 instance (`cba_instance_lightsail`) in the **default VPC** (`172.31.0.0/16`)
2. Set the security group inbound rules:

   | Type | Protocol | Port | Source |
   |---|---|---|---|
   | SSH | TCP | 22 | 0.0.0.0/0 |
   | HTTP | TCP | 80 | 172.26.0.0/16 |

   > Note: HTTP is already restricted to the Lightsail CIDR from the start.

3. Install Apache as in Task 1
4. Test peering is working by running this from inside the Lightsail terminal:

```bash
curl http://<EC2_PRIVATE_IP>:80
```

You should receive Apache HTML in response. If the connection times out, verify the peering connection is active in the VPC console and check the route tables.

---

### Task 6 — Lock Down the EC2 Security Group

With peering confirmed, update the EC2 security group to remove public HTTP access:

1. Open the EC2 console → **Security Groups**
2. Select `cba_security_group`
3. Edit inbound rules — change the HTTP rule source from `0.0.0.0/0` to `172.26.0.0/16`
4. Save

EC2 is now unreachable from the public internet on port 80. Only traffic originating from within the Lightsail VPC will be accepted.

---

### Task 7 — Install and Configure Nginx on Lightsail

**7.1 — Install Nginx**

SSH into the Lightsail instance and run:

```bash
sudo su
yum update -y
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
```

**7.2 — Configure reverse proxy**

Open the Nginx configuration file:

```bash
nano /etc/nginx/nginx.conf
```

Inside the existing `server { }` block, add the following `location` block:

```nginx
location / {
    proxy_pass http://<EC2_PRIVATE_IP>:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
```

Replace `<EC2_PRIVATE_IP>` with your EC2 instance's private IP address (e.g. `172.31.15.24`).

Test the configuration:

```bash
nginx -t
```

If you see `syntax is ok` and `test is successful`, reload Nginx:

```bash
systemctl reload nginx
```

---

### Task 8 — Validate the Architecture

Run all three checks to confirm everything is working correctly.

**Check 1 — Browser test**

Navigate to `http://<LIGHTSAIL_PUBLIC_IP>` in your browser. Your custom page (or the default Apache page) should load. If it loads, traffic has successfully travelled:

```
Browser → Lightsail (Nginx) → VPC Peering → EC2 (Apache) → response
```

**Check 2 — Apache access log**

On the EC2 instance, tail the Apache access log while making a request through the Lightsail IP:

```bash
sudo tail -f /var/log/httpd/access_log
```

The source IP in the log should show `172.26.x.x` — the Lightsail private IP range. If you see a public IP here, the proxy headers are not being forwarded correctly.

**Check 3 — EC2 direct access blocked**

Attempt to load the EC2 public IP directly in a browser:

```
http://<EC2_PUBLIC_IP>
```

This should time out with a connection error. If it loads, the security group rule is not correctly restricting port 80 to `172.26.0.0/16`.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| `curl` from Lightsail to EC2 times out | Peering not active or route table missing | Verify peering connection is `active` in VPC console; check route tables on both sides |
| Browser shows Nginx default page, not Apache | proxy_pass pointing to wrong IP or port | Confirm EC2 private IP in nginx.conf; test with `curl` from Lightsail terminal |
| Apache access log shows public IPs | Proxy headers not set | Verify `proxy_set_header` lines are present in nginx.conf and Nginx was reloaded |
| EC2 public IP still loads in browser | Security group not updated | Confirm HTTP inbound rule source is `172.26.0.0/16`, not `0.0.0.0/0` |
| `nginx -t` fails | Syntax error in nginx.conf | Check for typos in `proxy_pass` — must include `http://` prefix and no trailing slash issues |

---

## Key Concepts

**VPC Peering limitations with Lightsail**
Lightsail's automatic VPC peering only works with the AWS default VPC. Custom VPC peering with Lightsail requires Transit Gateway or keeping EC2 in the default VPC with proper subnet segmentation.

**Why security group CIDRs matter**
Calling a subnet "private" is a label, not a security control. Restricting EC2's HTTP inbound rule to `172.26.0.0/16` is what enforces privacy at the network layer. Without this rule, EC2 remains publicly accessible regardless of how it is named.

**Traceroute for debugging peering**
When `curl` times out silently, `traceroute` to the EC2 private IP from Lightsail will show where packets are dying — pointing to a routing issue rather than an application configuration problem.

**Bandwidth economics**
EC2 outbound data: ~$0.09/GB beyond the free tier. Lightsail $5/month: 1 TB included transfer. For sites pushing significant outbound traffic, the Lightsail front-end pattern pays for itself quickly and makes monthly costs predictable.

---

## What's Next

| Project | Description |
|---|---|
| HTTPS + custom domain | Route 53 for DNS, Certbot on Lightsail for Let's Encrypt SSL |
| Private RDS layer | MySQL in a private subnet, accessible only from EC2 |
| CloudFront in front | Global edge caching in front of Lightsail |
| Terraform rebuild | Full infrastructure as code for this architecture |
| High availability | Application Load Balancer with multiple EC2 instances |

---

## Project Files

```
.
├── README.md          # This lab guide
└── index.html         # Custom portfolio page served by Apache on EC2
```

---

## Related Labs

- [Lab 1 — EC2 Apache Web Server in a Custom VPC](../lab-01-ec2-apache-custom-vpc)

---

## Author

**Ayobami Edun** — Backend Engineer | Cloud Engineer  
[LinkedIn](https://www.linkedin.com/in/ayobami-edun/) · [GitHub](https://github.com/Ayooluwabami)

> *This lab was inspired by a comment from Muhammad Harith Omar on LinkedIn — proof that one sentence from the right person can become an entire architecture project.*
