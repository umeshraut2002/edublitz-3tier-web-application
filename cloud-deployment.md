---

# 🚀 EduBlitz 3-Tier Web Application on AWS

A beginner-friendly AWS project where:

**Students submit an enquiry form → Data is stored in Amazon RDS MySQL**

---

# 📦 Repository

GitHub Repository:
[https://github.com/atulyw/edublitz-3tier-web-application](https://github.com/atulyw/edublitz-3tier-web-application)

```bash
git clone https://github.com/atulyw/edublitz-3tier-web-application.git
cd edublitz-3tier-web-application
```

---

# 🏗 Architecture Overview

```
User
   ↓
CloudFront
   ↓
S3 (Static Website - HTML/CSS/JS)
   ↓
EC2 (Java Backend API - Port 8080)
   ↓
RDS MySQL (Private Subnet)
```

| Tier          | AWS Service | Role                   |
| ------------- | ----------- | ---------------------- |
| Web Tier      | S3          | Serves frontend        |
| CDN           | CloudFront  | Secure global delivery |
| App Tier      | EC2         | Java backend API       |
| Database Tier | RDS MySQL   | Stores enquiries       |

---

# 📁 Project Structure

```
edublitz-3tier-web-application/
│
├── frontend/
│   ├── index.html
│   ├── style.css
│   └── script.js
│
├── backend/
│   ├── App.java
│   ├── install.sh
│   └── schema.sql
│
└── README.md
```

---

# 🧱 SECTION 1 — Create VPC

1. Go to **VPC Dashboard**
2. Click **Create VPC**
3. Choose **VPC Only**
4. Name: `edublitz-vpc`
5. CIDR: `10.0.0.0/16`
6. Create

Then:

* Actions → Edit VPC settings
* ✅ Enable DNS hostnames

---

# 🌐 SECTION 2 — Create Subnets

### Public Subnet (EC2)

| Setting | Value                  |
| ------- | ---------------------- |
| Name    | edublitz-public-subnet |
| AZ      | ap-southeast-1a        |
| CIDR    | 10.0.1.0/24            |

Enable Auto-assign public IP.

---

### Private Subnets (RDS)

| Name               | AZ | CIDR        |
| ------------------ | -- | ----------- |
| edublitz-private1a | 1b | 10.0.2.0/24 |
| edublitz-private1b | 1c | 10.0.3.0/24 |

---

# 🌍 SECTION 3 — Internet Gateway

1. Create Internet Gateway → `edublitz-igw`
2. Attach to VPC

---

# 🛣 SECTION 4 — Route Tables

### Public Route Table

Add route:

```
Destination: 0.0.0.0/0
Target: Internet Gateway
```

Associate with public subnet.

---

### Private Route Table

Associate with both private subnets.

---

# 🔐 SECTION 5 — Security Groups

## EC2 App SG

Inbound:

| Type       | Port | Source    |
| ---------- | ---- | --------- |
| SSH        | 22   | My IP     |
| Custom TCP | 8080 | 0.0.0.0/0 |

---

## RDS SG

Inbound:

| Type  | Port | Source             |
| ----- | ---- | ------------------ |
| MySQL | 3306 | EC2 Security Group |

---

# 🗄 SECTION 6 — Create RDS MySQL

### Engine

MySQL 8.x

### DB Identifier

```
edublitz-db
```

### Master Username

```
admin
```

### Database Name

```
edublitz
```

### Subnet Group

Use private subnets.

### Security Group

Attach DB security group.

Wait until status = **Available**

Copy Endpoint:

```
edublitz-db.xxxxxx.ap-southeast-1.rds.amazonaws.com
```

---

# 💻 SECTION 7 — Launch EC2

### Settings

| Field     | Value           |
| --------- | --------------- |
| AMI       | Amazon Linux 2  |
| Instance  | t2.micro        |
| VPC       | edublitz-vpc    |
| Subnet    | public subnet   |
| Public IP | Enabled         |
| SG        | edublitz-app-sg |

---

## 📜 User Data Script (IMPORTANT)

Modify BEFORE pasting:

```bash
#!/bin/bash
set -e

cd /home/ec2-user

yum update -y
yum install -y git java-17-amazon-corretto mysql

git clone https://github.com/atulyw/edublitz-3tier-web-application.git

cp -r edublitz-3tier-web-application/backend/* /home/ec2-user/
chown -R ec2-user:ec2-user /home/ec2-user
chmod +x /home/ec2-user/install.sh

sudo -u ec2-user bash -c "
/home/ec2-user/install.sh \
  --db-host YOUR_RDS_ENDPOINT \
  --db-user YOUR_DB_USER \
  --db-password 'YOUR_DB_PASSWORD'
"

systemctl daemon-reload
systemctl enable edublitz-backend
systemctl restart edublitz-backend
```

---

# 🔎 Verify Backend

SSH into EC2:

```bash
sudo systemctl status edublitz-backend
```

Should show:

```
active (running)
```

Test:

```bash
curl http://localhost:8080
```

Expected:

```
EduBlitz 3-Tier Backend
```

Test insert:

```bash
curl -X POST http://localhost:8080/enquiry \
-d "name=Test&email=test@test.com&course=AWS&message=Hello"
```

---

# 🪣 SECTION 8 — Create S3 Frontend Bucket

1. Create bucket
2. Upload:

   * index.html
   * style.css
   * script.js

Before uploading script.js:

Replace:

```js
const BACKEND_URL = "http://YOUR_EC2_PUBLIC_IP:8080";
```

---

# 🌍 SECTION 9 — CloudFront Setup (CORRECT FLOW)

### Origin 1 — S3

Origin Domain:

```
your-bucket.s3.amazonaws.com
```

---

### Origin 2 — EC2 Backend

Origin Domain:

```
ec2-public-dns.amazonaws.com
```

Protocol: HTTP
Port: 8080

---

# 🔥 BEHAVIOR CONFIGURATION (IMPORTANT FIX)

## Default Behavior (*)

| Setting         | Value                  |
| --------------- | ---------------------- |
| Origin          | S3                     |
| Allowed Methods | GET, HEAD              |
| Viewer Protocol | Redirect HTTP to HTTPS |

---

## /enquiry* Behavior

| Setting               | Value                                        |
| --------------------- | -------------------------------------------- |
| Origin                | EC2                                          |
| Allowed Methods       | GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE |
| Cache Policy          | CachingDisabled                              |
| Origin Request Policy | AllViewer                                    |

---

## /enquiries* Behavior

Same as above.

---

# 🚨 Why POST Failed Earlier?

Because Default behavior allowed only:

```
GET, HEAD, OPTIONS
```

POST was blocked → 405 Method Not Allowed.

Fix = Create specific behavior for backend paths.

---

# 🧪 Final Test Flow

1. Open CloudFront URL
2. Fill form
3. Submit
4. Click Refresh
5. Data appears

---

# 🛠 Debug Guide (Production Method)

### Check backend logs

```bash
journalctl -u edublitz-backend -f
```

### Check DB connection

```bash
mysql -h <endpoint> -u admin -p
```

### Verify data

```sql
USE edublitz;
SELECT * FROM enquiries;
```

---

# ✅ What You Achieved

✔ VPC isolation
✔ Private RDS
✔ Secure DB access
✔ Public EC2 backend
✔ S3 frontend
✔ CloudFront CDN
✔ Proper POST routing
✔ Production-grade 3-tier architecture

---

# 🎯 Final Architecture

```
User
  ↓
CloudFront
  ↓
S3 (Frontend)
  ↓
EC2 (Java Backend)
  ↓
RDS MySQL (Private)
```

---
