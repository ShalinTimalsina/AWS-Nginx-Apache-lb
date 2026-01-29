# AWS Nginx Load Balancer with Two Apache Servers (Public + Private Subnet Architecture)

This is a production‑style, cost‑efficient AWS architecture that uses:

- **Nginx** as a reverse proxy + load balancer (public subnet)  
- **Two Apache servers** hosting web applications (private subnet)  
- **Elastic IP**, **Security Groups**, and **custom VPC**  
- **Round Robin load balancing**

This project demonstrates real-world **cloud architecture**, **Linux server administration**, **Nginx/Apache skills**, and **DevOps-level configuration understanding**.

---

## 🏗 Architecture Overview

Architecture Diagram/Nginx-Apache-Lb-Architecture-Diagram.png

### Components Used
- VPC (10.0.0.0/16)
- Public Subnet → Nginx Load Balancer
- Private Subnet → Apache App Servers
- Internet Gateway, Nat gateway + Route Tables
- Security Groups:
  - SG-For-Nginx → Allows HTTP/HTTPS from Internet
  - SG-For-APP → Allows port 8080 only from SG-For-Nginx
- Elastic IP for Nginx
- EC2 Instances
  - nginx-lb (public)
  - Apache-server-1 (private)
  - Apache-Server-2 (private)

---

##  Nginx Load Balancer Configuration

```nginx
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



