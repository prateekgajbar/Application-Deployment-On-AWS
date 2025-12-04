# Application Deployment on AWS | Three-Tier Architecture Project 

This project demonstrates the complete deployment of a real-world production-ready web application on AWS using a secure Three-Tier Architecture. The infrastructure is designed using best practices for security, scalability, and high availability.

This project is ideal for DevOps learners, Cloud Engineers, and AWS Beginners who want to understand real deployment workflow.

---
# Project Objective

To deploy a web application on AWS using:

- Secure networking using VPC
- Public and Private Subnets
- Backend and Database in private network
- EC Instance 

---

# Architecture Overview

- Frontend Layer (Public Subnet) – User-facing application
- Backend Layer (Private Subnet) – Business logic
- Database Layer (Private Subnet) – MySQL / RDS
- Internet Gateway – Public internet access
- NAT Gateway – Private subnet internet access

---

 # AWS Services Used

- Amazon EC2
- Amazon VPC
- Amazon S3
- Public & Private Subnets
- Internet Gateway
- NAT Gateway
- Route Tables
- Amazon RDS (MySQL)
- Security Groups


---

# Prerequisites

Before starting:

- AWS Account
- Basic Linux Knowledge
- AWS Console or AWS CLI
- EC2 Key Pair

This architecture follows a secure **3-tier AWS infrastructure**:
1. Public Layer (Nginx + Jump Server)
2. Private Application Layer (Tomcat Server)
3. Private Database Layer (RDS + DB-EC2)

This project demonstrates how I deployed a Java-based Student Application using industry best practices.

---

#  Architecture Overview

Below is the high-level architecture of the 3-tier deployment:

            +-----------------------------+
            |       Internet Users        |
            +--------------+--------------+
                           |
                      (Port 80)
                           |
                  +--------v---------+
                  |   Nginx Proxy    |  <-- Public Subnet
                  |   Jump Server    |
                  +--------+---------+
                           |
                   SSH + Reverse Proxy
                           |
                +----------v-----------+
                |   Tomcat App Server |  <-- Private Subnet-2
                |   (Java .war App)   |
                +----------+-----------+
                           |
                    JDBC Connection
                           |
                +----------v-----------+
                |     RDS MariaDB     |  <-- Private Subnet-3
                +----------------------+

This ensures:
- Application & DB server  remain private
- Only Proxy server has Public IP
- Secure SG-to-SG communication

---

# 1. VPC Setup
VPC CIDR: 172.25.0.0/16  
Region: ap-south-1 (Mumbai)

<img width="1920" height="819" alt="image" src="https://github.com/user-attachments/assets/55217701-9ede-42d2-bc36-6bcd25f7e231" />




The VPC provides isolated networking for all AWS resources.

---

# 2. Subnets (3 Total)

## Public Subnet (Proxy Server)
CIDR: 172.25.0.0/20  
Purpose:
- Nginx Reverse Proxy
- Proxy Server for SSH
  
## Private Subnet-1 (Application Layer)
CIDR: 172.25.16.0/20 
Purpose:
- Tomcat Java Application Server  
- No Public IP for security

## Private Subnet-2 (Database Layer)
CIDR: 172.25.32.0/20  
Purpose:
- RDS MariaDB  
- Backend Database EC2 (optional)  
- No Public IP

 <img width="1920" height="821" alt="image" src="https://github.com/user-attachments/assets/5c5490b7-431c-4bd1-87c6-2a28d71f287e" />


---

# 3. Internet Gateway
Created and attached to the VPC to allow internet access for the public subnet.

<img width="1920" height="818" alt="image" src="https://github.com/user-attachments/assets/857647f3-4711-43fa-a696-9809637e9c43" />


---

# 4. NAT Gateway 
Created inside Public Subnet  
Assigned an Elastic IP  
Provides **internet access to private subnets** for:
- OS updates
- Package installations
- Database client installations

<img width="1920" height="823" alt="image" src="https://github.com/user-attachments/assets/88da0dc7-3823-424d-8dac-dd74d5e019ee" />


---

# 5. Route Tables

## Public Route Table
Associated with:
- Public Subnet

Route:
0.0.0.0/0 → Internet Gateway

## Private Route Table
Associated with:
- Private Subnet-2 (App)
- Private Subnet-3 (DB)

Route:
0.0.0.0/0 → NAT Gateway

<img width="1920" height="824" alt="image" src="https://github.com/user-attachments/assets/79031431-a042-4ee4-9e31-5bdbd9ab1bd0" />


---

# 6. Security Group 

A single SG was used for simplicity.

## Inbound Rules
22 → My IP (SSH Access)  
80 → Anywhere (HTTP to Nginx)  
8080 → SG Itself (Tomcat internal communication)  
3306 → SG Itself (DB access from App Server)

<img width="1920" height="820" alt="image" src="https://github.com/user-attachments/assets/7eae361e-4303-481e-b980-bee202f7d1e7" />


## Outbound
ALL → Allowed

## Attached To:
- Proxy EC2
- App EC2
- Database EC2
- RDS MariaDB

Why one SG?  
Because using “SG-to-SG” traffic ensures only internal AWS resources communicate.

---

# 7. EC2 Instances

## 1️⃣ Proxy / Jump Server (Public Subnet)
Purpose:
- Reverse Proxy (Nginx)
- Entry point for all SSH connections
- Public IP enabled

## 2️⃣ Application Server (Tomcat)
- Hosted in Private Subnet-2  
- Runs Java JDK + Tomcat  
- No public exposure

## 3️⃣ Database EC2 
- Hosted in Private Subnet-3  
- Used to connect to RDS using CLI

---

# 8. Transfer SSH Key to Jump Server
scp -i my-Key.pem my-Key.pem ec2-user@3.110.186.184:/home/ec2-user/  
chmod 400 myKey.pem

<img width="1920" height="1023" alt="image" src="https://github.com/user-attachments/assets/59e7a427-8b4e-4caa-a30e-ec04809df441" />


---

# 9. SSH from Proxy → Private Instances
SSH into App Server:  
ssh -i myKey.pem ec2-user@App-Private-IP

SSH into Database EC2:  
ssh -i myKey.pem ec2-user@DB-Private-IP

This ensures **no direct SSH** into private servers.

---

# 10. Install Java & Tomcat on Application Server

sudo yum install java -y  
curl -O tomcat.tar.gz  
tar -xvzf tomcat.tar.gz -C /opt  
/opt/apache-tomcat/bin/catalina.sh start


<img width="1920" height="1026" alt="image" src="https://github.com/user-attachments/assets/00c4889a-bf5f-4066-b8e6-726f1988c69e" />

---

# 11. Deploy Java Application (.war)

cd /opt/apache-tomcat/webapps  
curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/student.war

<img width="1920" height="231" alt="image" src="https://github.com/user-attachments/assets/c74b0f70-71d9-4cf4-8320-52e930dcd22d" />


WAR file is auto extracted by Tomcat.

---

# 12. Create RDS MariaDB 10.5

Engine: MariaDB  
Version: 10.5  
Connectivity: Private Only  
Multi-AZ: Optional  
Backup retention: Enabled  
Security Group: Same SG as EC2

<img width="1920" height="1026" alt="image" src="https://github.com/user-attachments/assets/4d9bf16d-a51f-41d8-917d-4dc85c1eed80" />


Why RDS?  
- Auto backups  
- High availability  
- Better than hosting DB on EC2  
- Automatic patching

---

# 13. Connect to RDS from DB EC2

mysql -h RDS-endpoint -u admin -p

Create database:
CREATE DATABASE studentapp;  
USE studentapp;

CREATE TABLE if not exists students(student_id INT NOT NULL AUTO_INCREMENT,
	student_name VARCHAR(100) NOT NULL,
    student_addr VARCHAR(100) NOT NULL,
	student_age VARCHAR(3) NOT NULL,
	student_qual VARCHAR(20) NOT NULL,
	student_percent VARCHAR(10) NOT NULL,
	student_year_passed VARCHAR(10) NOT NULL,
 PRIMARY KEY (student_id)
	);

<img width="1920" height="1022" alt="image" src="https://github.com/user-attachments/assets/b4f888f0-cd28-4f3d-8d81-c2404be8e741" />


---

# 14. Add MySQL Connector to Tomcat

cd /opt/apache-tomcat/lib  
curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/mysql-connector.jar

<img width="1920" height="1024" alt="image" src="https://github.com/user-attachments/assets/8491725e-5367-434e-9b51-00db15694a7e" />



This enables Tomcat to connect to MySQL.

---

# 15. Tomcat DB Connection Setup

File to edit:
- /opt/apache-tomcat/conf/context.xml

Add resource:
Add:

<Resource name="jdbc/TestDB" auth="Container" type="javax.sql.DataSource"
username="admin" password="yourpassword"
driverClassName="com.mysql.jdbc.Driver"
url="jdbc:mysql://<RDS-endpoint>:3306/studentapp"

<img width="1920" height="1019" alt="image" src="https://github.com/user-attachments/assets/5590e234-5a8f-4242-a5d5-7197c049d902" />




Restart Tomcat:
- /opt/apache-tomcat/bin/catalina.sh stop  
- /opt/apache-tomcat/bin/catalina.sh start

<img width="1920" height="862" alt="image" src="https://github.com/user-attachments/assets/427002a0-8352-4ce2-93f6-3588ef26c24a" />


---

# 16. Configure Nginx Reverse Proxy

Install:
sudo yum install nginx -y  
sudo systemctl start nginx,
sudo systemctl enable nginx

<img width="1920" height="1023" alt="image" src="https://github.com/user-attachments/assets/3ce10749-7c41-4237-b31d-a66258b86f7e" />


Edit:
/etc/nginx/nginx.conf

Add:
location / {  
    proxy_pass http://App-Private-IP:8080/student/;  
}

<img width="1920" height="1025" alt="image" src="https://github.com/user-attachments/assets/0789815e-2c2f-4d10-9bf9-de695ed5ed70" />


Restart:
sudo systemctl restart nginx

Access:
http://Proxy-Public-IP

<img width="1920" height="1022" alt="image" src="https://github.com/user-attachments/assets/a8a168af-e999-4e82-8507-21b775370330" />

<img width="1920" height="976" alt="image" src="https://github.com/user-attachments/assets/c1810738-3b08-4fca-a9bf-151633956643" />




# 3-tier Application 

Users → Nginx Proxy → Tomcat App → RDS Database

All communication happens over secure private subnets, with controlled entry points.

---
