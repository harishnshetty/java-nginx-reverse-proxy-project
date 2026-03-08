# java-nginx-reverse-proxy-project
## For more projects, check out  
[https://harishnshetty.github.io/projects.html](https://harishnshetty.github.io/projects.html)

[![Video Tutorial](https://github.com/harishnshetty/image-data-project/blob/14a9dff8d061a153eeeb15b091e6c25b7bb00ee0/app1-java.png)](https://youtu.be/BgyYqUXuHuk?si=Gi6vkxhnVJQBILkG)

[![Channel Link](https://github.com/harishnshetty/image-data-project/blob/14a9dff8d061a153eeeb15b091e6c25b7bb00ee0/app2-java.png)](https://youtu.be/BgyYqUXuHuk?si=Gi6vkxhnVJQBILkG)


## 🏗️ Architecture Overview

```
User (Internet)
      │
      ▼  HTTP :80
┌─────────────────┐
│   Proxy Server  │  ← Public EC2 (Nginx)
│  (Nginx)        │      Security Group: Allow 80 from 0.0.0.0/0
└────────┬────────┘
         │ HTTP :8080 (internal only)
         ▼
┌─────────────────┐
│ Backend Server  │  ← Private EC2 (Java + Tomcat)
│ (Tomcat :8080)  │      Security Group: Allow 8080 ONLY from Proxy SG
└────────┬────────┘
         │ MySQL :3306 (internal only)
         ▼
┌─────────────────┐
│   Amazon RDS    │  ← MySQL Database
│   (MySQL)       │      Security Group: Allow 3306 ONLY from Backend SG
└─────────────────┘
```

---

## STEP 1 – Install Java 17 on Backend EC2 Ubuntu Machine

```bash
sudo apt update -y
sudo apt install -y openjdk-17-jdk

# Verify
java -version
```

---

## STEP 2 – Install Tomcat 9 on Backend EC2 Ubuntu Machine

[https://tomcat.apache.org/download-90.cgi](https://tomcat.apache.org/download-90.cgi)

```bash
cd /opt

# Download Tomcat 9
sudo wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.115/bin/apache-tomcat-9.0.115.tar.gz

# Extract and rename
sudo tar -xvzf apache-tomcat-9.0.115.tar.gz
sudo mv apache-tomcat-9.0.115 tomcat9


cd /opt/tomcat9/lib/
wget https://repo1.maven.org/maven2/mysql/mysql-connector-java/5.1.49/mysql-connector-java-5.1.49.jar

mv mysql-connector-java-5.1.49.jar mysql-connector-java.jar

# Set permissions
sudo chmod -R 755 /opt/tomcat9
```

```bash
cd /opt/tomcat/webapps/
sudo wget https://raw.githubusercontent.com/harishnshetty/java-nginx-reverse-proxy-project/refs/heads/main/student.war
```

```bash
# Start Tomcat
sudo /opt/tomcat9/bin/shutdown.sh
sudo /opt/tomcat9/bin/startup.sh

# Verify
curl http://localhost:8080
```

> ✅ You should see the Tomcat welcome HTML output.

---

## STEP 3 – Install MySQL Locally

```bash
sudo apt install -y mysql-server

# Start MySQL
sudo systemctl start mysql
sudo systemctl enable mysql
```
```bash
mysql -h studentdb-instance-1.cne0wymoyx30.ap-south-1.rds.amazonaws.com -u admin -p -e "CREATE DATABASE IF NOT EXISTS studentdb; USE studentdb; DROP TABLE IF EXISTS students; CREATE TABLE students (student_id INT NOT NULL AUTO_INCREMENT, student_name VARCHAR(100) NOT NULL, student_addr VARCHAR(255), student_age INT, student_qual VARCHAR(100),student_percent FLOAT, student_year_passed INT, PRIMARY KEY (student_id));"
# Enter password: rootroot
```

## STEP 4 – MYSQL AURORA DB SETUP


### Set the Context XML in the Backend EC2 Machine
```bash
sudo mv /opt/tomcat9/conf/context.xml /opt/tomcat9/conf/context.xml.bak
sudo nano /opt/tomcat9/conf/context.xml
```
```bash
<?xml version="1.0" encoding="UTF-8"?>
<Context>
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <WatchedResource>${catalina.base}/conf/web.xml</WatchedResource>
    <Resource name="jdbc/TestDB"
              auth="Container"
              type="javax.sql.DataSource"
              username="admin"
              password="rootroot"
              driverClassName="com.mysql.jdbc.Driver"
              url="jdbc:mysql://studentdb-instance-1.cne0wymoyx30.ap-south-1.rds.amazonaws.com:3306/studentdb"/>
</Context>
```


## Restart the tomcat
# Start Tomcat
```bash
sudo /opt/tomcat9/bin/shutdown.sh
sudo /opt/tomcat9/bin/startup.sh
```
## STEP 5 – Setup Nginx Reverse Proxy on Proxy EC2 in Amazon Linux

### SSH into Proxy EC2

```bash
ssh -i your-key.pem ec2-user@<Proxy-Public-IP>
```

### Install and Start Nginx

```bash
sudo yum update -y
sudo yum install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

### Configure Nginx as a Reverse Proxy

```bash
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
sudo nano /etc/nginx/nginx.conf
```

Inside the existing `server { }` block, add this `location` block:

```nginx
location /student/ {
    proxy_pass         http://<Backend-Private-IP>:8080/student/;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
}
```

```nginx
location /student/ {
    proxy_pass         http://172.31.11.73:8080/student/;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
}
```
>  Replace `<Backend-Private-IP>` with your backend EC2's **Private IP** (e.g., `172.31.31.215`).  
> Use the **private IP**, not the public IP, for internal communication.

### Test and Reload Nginx

```bash
# Test configuration syntax
sudo nginx -t

# Restart Nginx to apply changes
sudo systemctl restart nginx
```

```text
http://proxy-public-ip/student/viewStudents
```