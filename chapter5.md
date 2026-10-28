# ☁️ AWS EC2 — Quick Notes

### 1. What is EC2?

**EC2 = Elastic Compute Cloud**

👉 Virtual server in the AWS cloud.

Used to:

* Run applications / websites
* Run backend servers
* Install software (Apache, Node.js, etc.)

---

### 2. Launch EC2 Instance 🚀

**AWS Console → EC2 → Launch Instance**

**1. Name**
→ e.g. `my-first-ec2`

**2. AMI (OS)**
→ Amazon Machine Image = OS template

* Amazon Linux → user: `ec2-user`
* Ubuntu → user: `ubuntu`

**3. Instance Type**
→ CPU + RAM  
→ Free Tier: `t2.micro` / `t3.micro`

**4. Key Pair** 🔑
→ Needed for SSH / PuTTY

1. Create new key pair
2. Type: **RSA**
3. Format: `.pem` (convert later) or `.ppk` (PuTTY direct)
4. Download and save safely

⚠️ Private key is shown **only once**.

**5. Network + Security Group** 🔐

* Enable **Auto-assign public IP**
* Security Group = virtual firewall

Inbound rules:

* **SSH** → Port `22` → Source: **My IP**
* **HTTP** → Port `80` (optional)
* **HTTPS** → Port `443` (optional)

**6. Storage**
→ Keep default (usually 8 GB)

**7. User Data (optional)** ⭐
→ Startup script that runs on first boot

```bash
#!/bin/bash
yum install httpd -y
systemctl start httpd
```

**8. Launch**
→ Wait until **Running** + **2/2 checks passed**  
→ Copy **Public IPv4 address**

---

### 3. Connect with PuTTY (Windows) 🖥️

#### Install

1. Download from [https://www.putty.org](https://www.putty.org)
2. Install **PuTTY** + **PuTTYgen**

#### Convert `.pem` → `.ppk`

👉 Skip if you already have `.ppk`

1. Open **PuTTYgen** → **Load**
2. Select `.pem` (show All Files)
3. **Save private key** → save as `.ppk`

> PuTTY needs `.ppk`, not `.pem`

#### Connect

1. Open **PuTTY**
2. Host Name:

```text
ec2-user@<PUBLIC_IP>
```

(Ubuntu → `ubuntu@<PUBLIC_IP>`)

3. Port: `22` → Connection type: **SSH**
4. **Connection → SSH → Auth → Credentials**
5. Browse → select `.ppk`
6. Click **Open** → Accept → logged in ✅

#### Common Errors

* **Timed out** → SSH not open in Security Group / wrong IP
* **Server refused our key** → wrong `.ppk` or username
* **Permission denied** → wrong key or username

---

### 4. Instance States

```text
Pending → Running → Stopped
                  ↓
              Terminated
```

* **Running** → ON
* **Stopped** → OFF (can start again)
* **Terminated** → deleted forever

⚠️ **Terminate ≠ Stop**

---

### 5. Billing ⚠️

After practice → **Stop** or **Terminate** the instance.

> EBS volumes and Elastic IPs can also create charges.

---

# 🧠 One-Minute Revision

**EC2** → Virtual server  
**AMI** → OS template  
**Instance Type** → CPU + RAM  
**Key Pair** → `.pem` / `.ppk`  
**Security Group** → Firewall (Port 22 = SSH)  
**User Data** → Startup script  
**PuTTYgen** → `.pem` → `.ppk`  
**PuTTY Host** → `ec2-user@public-ip`  
**Stop** → temporary OFF  
**Terminate** → delete forever