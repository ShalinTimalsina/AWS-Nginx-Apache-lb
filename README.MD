# AWS Nginx Load Balancer with Two Apache Servers  
**Public + Private Subnet Architecture | Round-Robin Load Balancing**

---

##  Project Overview

This project demonstrates a **production-style AWS architecture** using a custom VPC where:

- **Nginx** acts as a **Reverse Proxy + Load Balancer**
- **Two Apache web servers** run in **private subnets**
- Traffic flows securely from **Internet → Nginx → Apache servers**
- **Round-Robin load balancing** distributes requests evenly
- Backend servers are **not publicly accessible**

This setup is intentionally designed for **hands-on practice**, **cloud networking understanding**, and **DevOps fundamentals**.

---

##  Objectives

- Understand **public vs private subnet architecture**
- Implement **Nginx reverse proxy & load balancing**
- Secure backend servers using **Security Group → Security Group rules**
- Practice **Linux server administration**
- Gain experience with **real AWS networking components**

---

## 🏗 Architecture Overview

📂 **Architecture Diagram:**  
`Architecture Diagram/Nginx-Apache-Lb-Architecture-Diagram.png`

### Traffic Flow

Internet
   ↓
Elastic IP
   ↓
Nginx Load Balancer (Public Subnet)
   ↓
Private IP Communication
   ↓
Apache App Server 1 (Private Subnet)
   ↓
Apache App Server 2 (Private Subnet)



---

## 🧱 AWS Resources Used

### Networking
- **VPC:** `10.0.0.0/16`
- **Public Subnet:** Nginx Load Balancer
- **Private Subnet:** Apache App Servers
- **Internet Gateway:** Internet access for Nginx
- **NAT Gateway:** Outbound internet access for private servers
- **Route Tables:** Separate public and private routing

### EC2 Instances

| Instance Name | Role | Subnet |
|--------------|------|--------|
| nginx-lb | Nginx Reverse Proxy & Load Balancer | Public |
| App-Server-1 | Apache Web Server | Private |
| App-Server-2 | Apache Web Server | Private |

- Instance Type: **t3.micro**
- OS: **Ubuntu**
- Elastic IP attached **only to Nginx**

📌 **Why Elastic IP?**  
Ensures the main entry point IP never changes, which is critical in real-world environments.

---

## 🔐 Security Group Design (Detailed)

Security Groups are used as the **primary security control at the instance level**.  
This architecture follows the **principle of least privilege** while allowing temporary administrative access for setup and maintenance.

---

### 🔹 SG-For-Nginx (Public / Bastion + Load Balancer)

This security group is attached to the **Nginx Load Balancer** instance, which also acts as a **temporary bastion host**.

#### Inbound Rules
- **HTTP (80)** → `0.0.0.0/0`  
  Allows public web traffic to reach the Nginx load balancer.

- **SSH (22)** → Trusted IP (or temporarily `0.0.0.0/0` during setup)  
  Used for administrative access and as a **bastion entry point** to reach private servers.

#### Outbound Rules
- **All traffic allowed**

📌 **Purpose:**
- Acts as the **only public-facing entry point**
- Serves as a **reverse proxy + load balancer**
- Temporarily functions as a **bastion host** for private server access

---

### 🔹 SG-For-APP (Private Apache Servers)

This security group is attached to **App-Server-1** and **App-Server-2**, which are placed in private subnets.

#### Inbound Rules
- **TCP 8080** → **SG-For-Nginx**  
  Allows web traffic **only from the Nginx load balancer**.

- **SSH (22)** → **SG-For-Nginx** *(Temporary)*  
  Used during setup to allow the bastion host (Nginx) to SSH into private servers.

#### Outbound Rules
- **All traffic allowed**

📌 **Purpose:**
- Prevents **direct internet access** to backend servers
- Ensures Apache servers only accept:
  - Application traffic from Nginx
  - Temporary administrative access from the bastion host

---

## 🌐 NAT Gateway Role in Security

Apache servers reside in **private subnets** and do **not have public IPs**.

- The **NAT Gateway** allows:
  - `apt update`
  - Package installation (Apache, security updates)
  - Outbound-only internet access

- The NAT Gateway **does NOT allow inbound traffic**

📌 **Security Benefit:**
- Private servers remain unreachable from the internet
- Only **outbound connections** are permitted

---

##  Temporary Bastion Access Clarification

During the initial setup phase:

- SSH access from **SG-For-Nginx → SG-For-APP** was **temporarily enabled**
- This allowed:
  - Apache installation
  - Port changes
  - Configuration updates

📌 **Post-Setup Best Practice:**
- Remove SSH (22) access from SG-For-APP
- Retain only:
  - TCP 8080 from SG-For-Nginx

This hardens the environment and aligns with **production security standards**.

---

##  Final Security Summary

- Public traffic → **Nginx only**
- Backend servers → **Private & protected**
- NAT Gateway → **Outbound-only internet access**
- SG-to-SG rules → **Zero trust between tiers**
- Bastion access → **Temporary and controlled**

This design closely mirrors **real-world AWS production architectures**.


## 🧭 Access Strategy (Bastion-Style)

- `nginx-lb` acts as a **bastion host**
- Apache servers have **only private IPs**
- SSH access to private servers is done **through Nginx**

---

## ⚙️ Apache Server Configuration  
*(Performed on App-Server-1 and App-Server-2)*


### 1️⃣ Update Package Lists & Install Apache

```bash
sudo apt update
sudo apt install apache2 -y
```
Explanation:
Updates package lists and installs the Apache web server.

---

2️⃣ Change Apache Port from 80 to 8080

```bash
sudo sed -i 's/80/8080/g' /etc/apache2/ports.conf
sudo sed -i 's/:80/:8080/g' /etc/apache2/sites-enabled/000-default.conf
```
Why this is needed:

   - Nginx listens on port 80

   - Apache servers run on port 8080

   - Prevents port conflict and enables proxying

---

3️⃣ Restart Apache

```bash
sudo systemctl restart apache2
```
---

4️⃣ Create Custom Index Page

App-Server-1
```bash
echo "<h1>App Server 1</h1>" | sudo tee /var/www/html/index.html
```
App-Server-2
```bash
echo "<h1>App Server 2</h1>" | sudo tee /var/www/html/index.html
```
Purpose:
Helps visually confirm round-robin load balancing.

---

⚖️ Nginx Load Balancer Configuration

1️⃣ Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```
---

2️⃣ Configure Nginx Reverse Proxy

📄 File: /etc/nginx/sites-available/default
```bash
upstream backend_servers {
    server 10.0.2.11:8080;
    server 10.0.2.12:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
Configuration Explanation

   - `upstream backend_servers` defines the Apache pool

   - Nginx uses Round Robin by default

   - `proxy_pass` forwards traffic to backend servers

   - `Headers` preserve client IP information
---

3️⃣ Test and Reload Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```
---

**Validation**

   - Open browser

   - Visit Elastic IP of Nginx

   - Refresh the page multiple times

You should see:

    App Server 1

    App Server 2

Alternating on refresh ✔



