### DevOps Tooling Website Solution

The website will display a dashboard of some of the most useful DevOps tools as listed below:
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

#### Project Requirement:
The project will use the following components:
- AWS as infrastructure
- Web Server with Red Hat Enterprise Linux 8
- DB Server with Ubuntu 20.04 and MySQL
- Storage solution: Red Hat Enterprise Linux 8 + NFS Server

####  Start Implementation

##### Step I1: Create NFS Server: 

* Launch an EC2 instance with RHEL Linux 8 Operating System

##### Configure LVM on the server: 
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
##### Step 2: Install NFS server, configure it to start on reboot and confirm it's running
```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```
![image](https://user-images.githubusercontent.com/58276505/172159382-7f99ff56-4ed6-4a21-aceb-5a723add3c9b.png)

##### Step 3: Change permissions on NFS server to allow web servers to read, write and execute: 
```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service

##### Step 4: Export the mounts for webservers’ subnet cidr to connect as clients
Find the subnet CIDR of the NFS server and allow access to clients within the same subnet:
```
sudo vi /etc/exports

/mnt/apps <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/logs <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)
/mnt/opt <Subnet-CIDR>(rw,sync,no_all_squash,no_root_squash)

Esc + :wq!
sudo exportfs -arv
```
![image](https://user-images.githubusercontent.com/58276505/172163247-063660d7-874f-41b7-b8e0-08052a5cb1f2.png)
 
```
sudo systemctl restart nfs-server.service
```
##### Step 5: Check the port currently used by NFS and also edit the security group rules to allow inbound traffic on TCP 111, UDP 111, UDP 2049
```
rpcinfo -p | grep nfs
```

 #### Step II6: Creating and configuring the Database Server

##### Step7: Launch an EC2 instance of type Ubuntu 20.04 and install MySQL

* Install MySQL server:
```
sudo apt install mysql-server -y

sudo mysql_secure_installation
```
* Log into MySQL and create a database called 'tooling'

* Next, create a database user called webaccess and grant permissions
* Open and edit the bind address in /etc/mysql/mysql.conf.d/mysqld.cnf file to grant access to the 'webaccess' user from the webservers.

#### Step 8: Create and configure three identical webservers

##### Create three EC2 instances with Red Hat Linux 8 operating system

- Install NFS client 
```
sudo yum install nfs-utils nfs4-acl-tools -y
```
- Create a directory /var/www and mount it to target the NFS server’s export for apps
```
sudo mkdir /var/www
sudo mount -t nfs -o rw,nosuid 172.31.33.30:/mnt/apps /var/www
  
Add the following entry to /etc/fstab file so the changes persist:
```
172.31.33.30:/mnt/apps /var/www nfs defaults 0 0
```
Install Apache:
```
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```
![project 7d](https://user-images.githubusercontent.com/41236641/130780750-74c83432-4f29-4f2c-942f-5b31a16a1ae5.JPG)
- Check the /var/html folder to verify that the Apache files and directories are present. Check the /mnt/apps directory on NFS server to verify if the same Apache files and directories are present.

- Mount the log folder for Apache on webserver to NFS folder for /mnt/logs with same steps as above. Update the /etc/fstab and make sure changes persist.
- Install git and clone the tooling source code at https://github.com/darey-io/tooling.git
``` 
sudo yum install git -y
git clone https://github.com/darey-io/tooling.git
```
###### Deploy the tooling website’s code to the Webserver. Ensure that the html folder from the repository is deployed to /var/www/html

 #####  Update the website’s configuration to connect to the database

 ##### Create in MySQL a new admin user with username: myuser and password: password: with other database insert fields specified

 ##### Confirm the database can be accessed: 3.144.131.56/login.php
![project 7fdbcomplete](https://user-images.githubusercontent.com/41236641/130782769-aa84cffe-bc66-4efa-a59e-af7dc045257b.JPG)

 ##### Confirm the website is up from the webservers public ip address/index.php and can be accessed.
 
![Project 7esitecomplete](https://user-images.githubusercontent.com/41236641/130782970-2eabf4a0-199f-4a22-bcbc-3e58563969a9.JPG)
