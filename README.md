#  Application Deployment on AWS (3-Tier Architecture)
A complete production-style deployment of a Java Web Application using:
VPC • 3 Subnets • EC2 • RDS MariaDB 10.5 • Apache Tomcat • Nginx Reverse Proxy • Jump Server • NAT Gateway

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
- Application & DB remain private
- Public access only through Nginx load balancer
- Only Proxy has Public IP
- Secure SG-to-SG communication

---

# 1. VPC Setup
VPC CIDR: 172.25.0.0/16  
Region: ap-south-1 (Mumbai)

<img width="1920" height="819" alt="image" src="https://github.com/user-attachments/assets/55217701-9ede-42d2-bc36-6bcd25f7e231" />




The VPC provides isolated networking for all AWS resources.

---

# 2. Subnets (3 Total)

## Public Subnet (Jump + Proxy)
CIDR: 172.25.0.0/20  
Purpose:
- Nginx Reverse Proxy
- Jump Server for SSH
- NAT Gateway lives in this subnet

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

# 4. NAT Gateway (Highly Recommended)
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

# 6. Security Group Design (Professional)

A single consolidated SG was used for simplicity.

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


Best Practice:
- Create a dedicated tomcat user
- Configure systemd service for Tomcat

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

Create table:
CREATE TABLE students(  
student_id INT AUTO_INCREMENT PRIMARY KEY,  
student_name VARCHAR(100),  
student_addr VARCHAR(100),  
student_age VARCHAR(3),  
student_qual VARCHAR(20),  
student_percent VARCHAR(10),  
student_year_passed VARCHAR(10)  
);

---

# 14. Add MySQL Connector to Tomcat

cd /opt/apache-tomcat/lib  
curl -O https://s3-us-west-2.amazonaws.com/studentapi-cit/mysql-connector.jar

<img width="1905" height="196" alt="image" src="https://github.com/user-attachments/assets/3c0dacfc-d347-4570-96e2-6556c6a4136b" />


This enables Tomcat to connect to MySQL.

---

# 15. Tomcat DB Connection Setup

File to edit:
- /opt/apache-tomcat/conf/context.xml

Add resource:

<Resource name="jdbc/TestDB" auth="Container"  
type="javax.sql.DataSource"  
username="admin" password="yourpassword"  
driverClassName="com.mysql.jdbc.Driver"  
url="jdbc:mysql://RDS-endpoint:3306/studentapp"/>

Restart Tomcat:
- /opt/apache-tomcat/bin/catalina.sh stop  
- /opt/apache-tomcat/bin/catalina.sh start

---

# 16. Configure Nginx Reverse Proxy

Install:
sudo yum install nginx -y  
sudo systemctl start nginx

Edit:
/etc/nginx/nginx.conf

Add:
location / {  
    proxy_pass http://App-Private-IP:8080/student/;  
}

Restart:
sudo systemctl restart nginx

Access:
http://Proxy-Public-IP


# 3-tier Application 

Users → Nginx Proxy → Tomcat App → RDS Database

All communication happens over secure private subnets, with controlled entry points.

---
