## DevOps Tooling Website Solution

* The website will display a dashboard of some of the most useful DevOps tools as listed below:
- Jenkins
- Kubernetes
- JFrog Artifactory
- Rancher
- Docker
- Grafana
- Prometheus
- Kibana

### Architecture Of Project / Solution:

![image](https://user-images.githubusercontent.com/58276505/172155502-f8f9bb0c-432f-46a2-ae99-acd8bc6fc4b4.png)

#### Requirement for Project:
- AWS as infrastructure
- Web Server with Red Hat Enterprise Linux 8
- DB Server with Ubuntu 20.04 and MySQL
- Storage solution: Red Hat Enterprise Linux 8 + NFS Server

## Project Implementation

### Scetion A: Create NFS Server: 

* Launch an EC2 instance with RHEL Linux 8 Operating System

#### Configure LVM on the server: 
* Create three logical volumes of 'xfs' format and name them as:

```
     - lv-opt,
     - lv-apps and
     - lv-logs
```

* Create three mount points for the logical volumes on the { /mnt} directory: 

```
     - lv-apps mounted on /mnt/apps for webservers
     - lv-logs mounted on /mnt/logs for webserver logs
     - lv-opt mounted on  /mnt/opt for jenkins server
```

#### Step 2: Install NFS server, configure it to start on reboot and confirm it's running

```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```

![image](https://user-images.githubusercontent.com/58276505/172159382-7f99ff56-4ed6-4a21-aceb-5a723add3c9b.png)

#### Step 3: Change permissions on NFS server to allow web servers to read, write and execute: 

```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

##### Step 4: Export the mounts for webserver's subnet cidr to connect as clients.

* Find the subnet CIDR of the NFS server and allow access to clients within the same subnet

```
sudo vi /etc/exports
/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!
sudo exportfs -arv
```

![image](https://user-images.githubusercontent.com/58276505/172167643-f5658a61-9eda-407d-99b1-081d28e220d9.png)
 
* sudo systemctl restart nfs-server.service

#### Step 5: Check the port currently used by NFS and also edit the security group rules to allow inbound traffic on TCP 111, UDP 111, UDP 2049

![image](https://user-images.githubusercontent.com/58276505/172173281-a217ca0e-125b-477a-96ac-2f613315e5a0.png)

```
rpcinfo -p | grep nfs
```

### Section II: Launch ubuntu 20.04, Instal and Configure DB Server

- Launch an Ubuntu 20.04 instance and install MySQL

```
sudo apt-get update && sudo apt-get upgrade
sudo apt install mysql-server -y
sudo systemctl start mysql
sudo systemctl enable mysql
```

![image](https://user-images.githubusercontent.com/58276505/172187937-3a89218c-5cda-4e30-9602-24ad0d6bf6ba.png)

* Configure mysql to enable connection from any address:

```
 sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf
 Locate bind_address and change from 127.0.0.1 to 0.0.0.0
```

![image](https://user-images.githubusercontent.com/58276505/172188751-518fbc6c-b17c-4f64-9839-de7a184e3f85.png)

### Connect to mysql and create tooling db

```
sudo mysql
CREATE DATABASE tooling;
CREATE USER 'webaccess'@'webserver-CIDR' IDENTIFIED BY 'admin';
GRANT ALL PRIVILEGES ON tooling.* TO 'webaccess'@'webserver-CIDR' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

### Section III: Launch & Configure Web Servers(x3-server)

* Launch a new RHEL 8 instance and Install nfs client.
* Create /var/www directory and mount the NFS server's to /mnt/apps (remember to change: NFS-Server-Private-IP-Address)
* Edit /etc/fstab to ensure changes persist after reboot (add line for 3 webservers: Web-Server-Private-IP-Address:/mnt/apps /var/www nfs defaults 0 0)

```
sudo yum install nfs-utils nfs4-acl-tools -y
sudo mkdir /var/www 
sudo mount -t nfs -o rw,nosuid NFS-Server-Private-IP-Address:/mnt/apps /var/www
sudo mount -t nfs -o rw,nosuid NFS-Server-Private-IP-Address:/mnt/logs /var/www
#sudo mount -t nfs -o rw,nosuid NFS-Server-Private-IP-Address:/mnt/opt /var/www (verify)

sudo vi /etc/fstab     # (add line for 3 webservers: Web-Server-Private-IP-Address:/mnt/apps /var/www nfs defaults 0 0)
```

#### Image here (fstab.jpg)

* restart nfs (sudo systemctl start nfs)
* Install Apache (Run => sudo yum install httpd -y && sudo systemctl start httpd && sudo systemctl enable httpd && sudo systemctl status httpd )

![project 7d](https://user-images.githubusercontent.com/41236641/130780750-74c83432-4f29-4f2c-942f-5b31a16a1ae5.JPG)

* Check the /var/html folder to verify that the Apache files and directories are present and also check the /mnt/apps directory on NFS server to verify if the same Apache files and directories are present.

#### Fork the tooling repo to your GitHub and clone to the web servers. Copy the html folder to /var/www/html.
* Install git(Run => sudo yum install git -y)
* clone the tooling source code at https://github.com/darey-io/tooling.git (Run => git clone https://github.com/darey-io/tooling.git)

#### Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html

#### Install and configure PHP
- Install EPEL Repo (Run => sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm)
- Install yum utils and enable remi-repository (Run => sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm)
- Install PHP (Run => sudo dnf install php php-opcache php-gd php-curl php-mysqlnd)
- Start the php-fpm service and enable on boot-up (Run => sudo systemctl start php-fpm && sudo systemctl enable php-fpm)
- Instruct SELinux to allow Apache to execute the PHP code via PHP-FPM (Run => sudo setsebool -P httpd_execmem 1)

- Set SELinux Enforcing to 0 (Run => sudo setenforce 0)
- Update functions.php file with database configuration details

#### Image here![](functions.jpg)

 #####  Update the website’s configuration to connect to the database
 ##### Create in MySQL a new admin user with username: myuser and password: password: with other database insert fields specified
 ##### Confirm the database can be accessed: 3.144.131.56/login.php
 
* Apply tooling-db.sql script (mysql -h db_ip -u webaccess -p tooling < tooling-db.sql)
* Insert new user into the database (You can also do this by editing the tooling-db.sql script appropriately.) (INSERT INTO ‘users’ (’id’, ‘username’, ‘password’, ‘email’, ‘user_type’, ‘status’) VALUES -> (1, ‘myuser’, ‘5f4dcc3b5aa765d61d8327deb882cf99’, ‘user@mail.com’, ‘admin’, ‘1’);)
* Login to website using one of the webservers ip address

![project 7fdbcomplete](https://user-images.githubusercontent.com/41236641/130782769-aa84cffe-bc66-4efa-a59e-af7dc045257b.JPG)

 ##### Confirm the website is up from the webservers public ip address/index.php and can be accessed.
 
![Project 7esitecomplete](https://user-images.githubusercontent.com/41236641/130782970-2eabf4a0-199f-4a22-bcbc-3e58563969a9.JPG)
