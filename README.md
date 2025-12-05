# 🎯📚 Add Book+ Project — Simplifying Book Management

<img width="1273" height="658" alt="Untitled Diagram drawio (1)" src="https://github.com/user-attachments/assets/8ec3c525-64c1-4d45-a2bb-c7fd54091208" />


# 📚 Infrastructure Setup

This guide walks through the complete setup of the **BOOK+ Project** on AWS, including VPC, subnets, route tables, NAT gateway, security groups, EC2 instances, RDS, backend, frontend, load balancers, and Route53.

---

## 🏗️ 1. Create VPC
- **Name:** `project-vpc`  
- **CIDR:** `10.0.0.0/16`

---

## 🌐 2. Create Internet Gateway
- **Name:** `project-ig`  
- Attach to `project-vpc`

---

## 📦 3. Create Subnets
- **Public Subnets (Bastion):**
  - `public-subnet-1a` → `10.0.1.0/24`
  - `public-subnet-1b` → `10.0.2.0/24`

- **Frontend Private Subnets:**
  - `frontend-pvt-subnet-1a` → `10.0.3.0/24`
  - `frontend-pvt-subnet-1b` → `10.0.4.0/24`

- **Backend Private Subnets:**
  - `backend-pvt-subnet-1a` → `10.0.5.0/24`
  - `backend-pvt-subnet-1b` → `10.0.6.0/24`

- **RDS Subnets (Database):**
  - `RDS-subnet-1a` → `10.0.7.0/24`
  - `RDS-subnet-1b` → `10.0.8.0/24`

---

## 🛣️ 4. Create Route Table (Public)
- **Name:** `public-rt`  
- **VPC:** `project-vpc`  
- **Routes:**
  - `0.0.0.0/0` → `project-ig`  
- **Subnet Association:**
  - `public-subnet-1a`
  - `public-subnet-1b`

---

## 🔑 5. Create NAT Gateway
- **Name:** `project-nat`  
- **Subnet:** `public-subnet-1a`  
- **Elastic IP:** Allocated

---

## 🛣️ 6. Create Route Table (Private via NAT)
- **Name:** `private-rt-nat`  
- **VPC:** `project-vpc`  
- **Routes:**
  - `0.0.0.0/0` → `project-nat`  
- **Subnet Association:**
  - `frontend-pvt-subnet-1a`, `frontend-pvt-subnet-1b`
  - `backend-pvt-subnet-1a`, `backend-pvt-subnet-1b`
  - `RDS-subnet-1a`, `RDS-subnet-1b`

---

## 🔒 7. Create Security Groups
- **Public SG (`public-sg`):**
  - Inbound: All traffic `0.0.0.0/0`

- **Frontend SG (`frontend-sg`):**
  - Inbound: SSH `0.0.0.0/0`, HTTP via ALB SG

- **Backend SG (`backend-sg`):**
  - Inbound: SSH `22`, HTTP `80`, All traffic `0.0.0.0/0`

- **Load Balancer SG (`ALB-sg`):**
  - Inbound: HTTP `80`, HTTPS `443`, All traffic `0.0.0.0/0`

- **Database SG (`database-sg`):**
  - Inbound: MySQL/Aurora `3306` `0.0.0.0/0`

---

## 💻 8. Launch Instances
- **Bastion (Public)**  
- **Frontend (Private)**  
- **Backend (Private)** → requires keypair for SSH

---

## 🗄️ 9. Create RDS
- **Subnet Group:** `rds-sg` → `RDS-subnet-1a`, `RDS-subnet-1b`  
- **Database:** MySQL  
- **Password:** `chandan#1234`  
- **VPC:** `project-vpc`  
- **Public Access:** No  
- **AZs:** `1a, 1b`

---

# ⚙️ Backend Setup

## 🔗 10. Connect Backend via Bastion
```bash
sudo su -
vi <keypair>.pem
chmod 400 <keypair>.pem
ssh -i <keypair>.pem ec2-user@<backend-server>
```

- Install Git & clone repo:
```bash
yum install git -y
git clone https://github.com/CloudTechDevOps/2nd10WeeksofCloudOps-main.git
cd 2nd10WeeksofCloudOps-main/backend
```

- Configure `.env`:
```env
DB_HOST=<rds-endpoint>
DB_USERNAME=admin
DB_PASSWORD="chandan#1234"
PORT=3306
```

- Install DB & run schema:
```bash
yum install mariadb105-server -y
mysql -h <rds-endpoint> -u admin -p<password> < test.sql
```

- Install Node.js & run app:
```bash
sudo dnf install -y nodejs
npm install
npm install -g pm2
pm2 start index.js --name node-app
curl http://localhost
```

---

## 🎯 11. Create Target Group (Backend)
- **Name:** `backend-tg`  
- **Protocol:** HTTP  
- **Port:** 80  
- Register backend server

---

## 🌐 12. Create Load Balancer (Backend)
- **Name:** `bastion-ALB`  
- **Subnets:** `public-subnet-1a`, `public-subnet-1b`  
- **SG:** `ALB-sg`  
- **TG:** `backend-tg`  
- **AZs:** `1a, 1b`

---

# 🎨 Frontend Setup

## 🔗 13. Connect Frontend via Bastion
```bash
sudo su -
vi <keypair>.pem
chmod 400 <keypair>.pem
ssh -i <keypair>.pem ec2-user@<frontend-server>
```

- Install Apache:
```bash
yum install httpd -y
systemctl start httpd
systemctl enable httpd
```

- Install Node.js:
```bash
sudo dnf install -y nodejs
```

- Clone repo & configure:
```bash
git clone https://github.com/CloudTechDevOps/2nd10WeeksofCloudOps-main.git
cd 2nd10WeeksofCloudOps-main/client/src/pages
vi config.js
```

Change:
```js
const API_BASE_URL = "http://backend-loadbalancer-url";
```

- Build frontend:
```bash
cd client
npm install
npm run build
sudo cp -r build/* /var/www/html
```

---

## 🎯 16. Create Target Group (Frontend)
- **Name:** `frontend-tg`  
- **Protocol:** HTTP  
- **Port:** 80  
- Register frontend server

---

## 🌐 17. Create Load Balancer (Frontend)
- **Name:** `backend-ALB`  
- **Subnets:** `public-subnet-1a`, `public-subnet-1b`  
- **SG:** `ALB-sg`, `backend-sg`  
- **TG:** `frontend-tg`  
- **AZs:** `1a, 1b`

---

## 🌍 18. Configure Route53
- Domain: `chintu.shop`  
- Create alias record → Application Load Balancer → Region `us-east-1` → Select **Frontend LB**  
- Create alias record → Application Load Balancer → Region `us-east-1` → Select **Backend LB**

---

# ✅ Final Step
Open your domain in the browser → **BOOK+ Project is live!** 🎉

<img width="1920" height="1080" alt="Screenshot (251)" src="https://github.com/user-attachments/assets/b1089574-244d-480e-847b-a093d92f22d1" />



---
##  Using Docker Deploy Book Peoject

<img width="1200" height="633" alt="image" src="https://github.com/user-attachments/assets/25e8e5d2-1775-44a6-8005-04b13a3446ef" />




---

## 📦 Stack Overview

- **Frontend:** React/Node app served via Nginx container  
- **Backend:** API service container (Flask/Spring Boot depending on repo)  
- **Database:** MariaDB 10.5 installed directly on EC2 host  
- **Tools:** Docker, Git, EC2 (Amazon Linux/RHEL)

---

## 🖼️ Architecture Diagram

```mermaid
flowchart LR
  User((Browser)) -->|HTTP :83| Frontend[Frontend Container]
  Frontend -->|HTTP :84| Backend[Backend Container]
  Backend -->|TCP :3306| DB[(MariaDB 10.5)]
  subgraph EC2 Instance
    Frontend
    Backend
    DB
  end
```

---

## ⚙️ Prerequisites

- EC2 instance running Amazon Linux 2 or RHEL  
- Security Group inbound rules: open **ports 83 and 84** for HTTP access  
- SSH access to the instance  

---

## 🛠️ Step-by-Step Setup

### 1. Switch to root and install dependencies

```bash
sudo su -
yum install docker -y
yum install mariadb105-server -y
yum install git -y
```

### 2. Start Docker and verify

```bash
systemctl start docker
systemctl status docker
docker --version
```

### 3. Start and secure MariaDB

```bash
systemctl start mariadb
systemctl enable mariadb
mysql_secure_installation
```

Create database and user:

```sql
CREATE DATABASE appdb;
CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'StrongPassword123!';
GRANT ALL PRIVILEGES ON appdb.* TO 'appuser'@'localhost';
FLUSH PRIVILEGES;
```

---

### 4. Clone the repository

```bash
git clone https://github.com/chintu-cloud/3-tier-architecture.git
cd 3-tier-architecture
```

---

### 5. Configure and run Backend

```bash
cd backend
vi .env
```

Update `.env` with:

```env
DATABASE_URL=mysql://appuser:StrongPassword123!@127.0.0.1:3306/appdb
DATABASE_PASSWORD=StrongPassword123!
```

Build and run backend:

```bash
docker build -t backend .
docker run -dt --env-file .env -p 84:80 backend
docker ps
```

---

### 6. Configure and run Frontend

```bash
cd ../client/src/pages
vi config.js
```

Update `config.js` with backend URL:

```js
const BACKEND_BASE_URL = "http://<EC2_PUBLIC_IP>:84";
export default BACKEND_BASE_URL;
```

Build and run frontend:

```bash
cd ../../
docker build -t frontend .
docker run -dt -p 83:80 frontend
docker ps
```

---

## 🌐 Verification

- **Frontend:** `http://<EC2_PUBLIC_IP>:83`  
- **Backend:** `http://<EC2_PUBLIC_IP>:84`  

Example:  
```
Frontend → http://172.132.39.73:83
Backend  → http://172.132.39.73:84
```
<img width="1920" height="1080" alt="Screenshot (470)" src="https://github.com/user-attachments/assets/23b746a7-3e87-4167-9da0-7a9fc7d7cab7" />

---

## 🔧 Troubleshooting

- **Ports not reachable:** Check EC2 Security Group and firewall rules.  
- **Backend DB errors:** Confirm MariaDB is running and credentials match `.env`.  
- **Frontend not updated:** Rebuild image after editing `config.js`.  

---

## 🧹 Cleanup

```bash
docker ps
docker stop <container_id>
docker rm <container_id>
docker rmi frontend backend
systemctl stop docker
systemctl stop mariadb
```

---


                                                    
                                                  




