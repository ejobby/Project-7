# Project-7

# DEVOPS TOOLING WEBSITE SOLUTION

For this Project, 5 instances will be created on the AWS EC2. 4 instances of Red Hat Server (One for the NFS server and the other three for Webserver) and 1 instance of Ubuntu server for the Database Server.

Three EBS Volumes also be created in the same availability zone with the NFS Server and named NFS1, NFS2 and NFS3. The three EBS volumes will be attached to the NFS Server 

![Volume Creation](./Images/volume-add.png)

## 1. Prepare NFS Server

Launch the NFS Server Instance and connect to it via the terminal

Check if the volume attached are connected

`lsblk`

`df -h`

Then Create partition on the three Volumes that are named xvdf, xvdg, xvdh

`sudo gdisk xvdf`

![Partition Created](./Images/partitition.png)

Do same for xvdg and xvdh

Then Install LVM

`sudo yum update -y`

`sudo yum install lvm2 -y`

Use pvcreate utility to mark each of 3 disks as physical volumes (PVs) to be used by LVM

`sudo pvcreate /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`

`sudo pvs`

Use vgcreate utility to add all 3 PVs to a volume group (VG). Name the VG nfsdata-vg

`sudo vgcreate nfsdata-vg /dev/xvdf1 /dev/xvdg1 /dev/xvdh1`

`sudo vgs`

Use lvcreate utility to create 3 logical volumes. apps-lv, logs-lv and opt-lv


`sudo lvcreate -n apps-lv -L 9G nfsdata-vg`

`sudo lvcreate -n logs-lv -L 9G nfsdata-vg`

`sudo lvcreate -n opt-lv -L 9G nfsdata-vg`

`sudo lvs`

Use mkfs.xfs to format the logical volumes with xfs filesystem

`sudo mkfs.xfs /dev/nfsdata-vg/apps-lv`

`sudo mkfs.xfs /dev/nfsdata-vg/logs-lv`

`sudo mkfs.xfs /dev/nfsdata-vg/opt-lv`

![xfs filesystem creation](./Images/filesystem.png)

Create the following directory /mnt/apps, /mnt/logs, /mnt/opt and mount the filesystem

`sudo mkdir /mnt/apps`

`sudo mkdir /mnt/logs`

`sudo mkdir /mnt/opt`

Then mount the file system

`sudo mount /dev/nfsdata-vg/apps-lv /mnt/apps`

`sudo mount /dev/nfsdata-vg/logs-lv /mnt/logs`

`sudo mount /dev/nfsdata-vg/opt-lv /mnt/opt`

check if the mounting was successful 

`df -h`

![Mounting](./Images/mounting.png)

Then update the /etc/fstab with the volumes

`sudo vi /etc/fstab`

![Mounting permanent](./Images/fstab.png)

`sudo mount -a`

`sudo systemctl daemon-reload`

### Install the NFS Server

Copy the code line by line and run it to make nfs server installed and start automatically in case of any reboot

```
sudo yum -y update
sudo yum install nfs-utils -y
sudo systemctl start nfs-server.service
sudo systemctl enable nfs-server.service
sudo systemctl status nfs-server.service
```
### Then we set up permission that will allow our Web servers to read, write and execute files on NFS.

Copy the line after each other and run it 

```
sudo chown -R nobody: /mnt/apps
sudo chown -R nobody: /mnt/logs
sudo chown -R nobody: /mnt/opt

sudo chmod -R 777 /mnt/apps
sudo chmod -R 777 /mnt/logs
sudo chmod -R 777 /mnt/opt

sudo systemctl restart nfs-server.service
```

Configure access to NFS for clients within the same subnet

`sudo vi /etc/exports`

Then insert the codes below inside the file

```
/mnt/apps 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/logs 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)
/mnt/opt 172.31.32.0/20(rw,sync,no_all_squash,no_root_squash)
```
`sudo exportfs -arv`

### Check which port is used by NFS and open it using Security Groups (add new Inbound Rule)

`rpcinfo -p | grep nfs `

Then on the AWS security group, make the necessary rule for the connection

![MSecurity Rule](./Images/rule.png)

## 2. Configure the Database Server

Connect to the Ubuntu Instance already tagged as the Database server

`sudo apt update`

`sudo apt install mysql-server`

`sudo mysql`

Then copy the code line by line and run it

```
create database tooling;

create user 'webaccess'@'172.31.32.0/20` identified by 'password'

grant all privileges on tooling.* to 'webaccess'@'172.31.32.0/20`;

flush privileges;

show databases;

exit
```

Edit the mysqld.cnf to allow access from the webserver

`sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf`

change the bind address to 0.0.0.0

![Binding Address](./Images/bind.png)

Then on the AWS EC2, add another inboundrule to allow MYSQL ports

![Inbound Rule](./Images/mysql-port.png)
## 3. Prepare the Web Servers

Connect to one of the Red hat server already tagged Webserver

Install nfs client on the server

`sudo yum install nfs-utils nfs4-acl-tools -y`

Mount /var/www/ and target the NFS server’s export for apps

`sudo mkdir /var/www`

`sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/apps /var/www`

Update the /etc/fstab to make sure that the changes will persist on Web Server after reboot

`sudo vi /etc/fstab`

```
<NFS-Server-Private-IP-Address>:/mnt/apps /var/www nfs defaults 0 0
```

### Install Remi’s repository, Apache and PHP

Copy the code below and run it line by line

```
sudo yum install httpd -y

sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm

sudo dnf install dnf-utils http://rpms.remirepo.net/enterprise/remi-release-8.rpm

sudo dnf module reset php

sudo dnf module enable php:remi-7.4

sudo dnf install php php-opcache php-gd php-curl php-mysqlnd

sudo systemctl start php-fpm

sudo systemctl enable php-fpm

sudo setsebool -P httpd_execmem 1
```

Verify if the /mnt/apps on nfs server and /var/www on webserver have synchronised. 

On the NFS SERVER

`sudo ls /mnt/apps`

![mnt-app-verify](./Images/mnt-apps-verify.png)

On the WebServer

`sudo ls /var/www`

![var-www-verify](./Images/var-www-verify.png)

Locate the log folder for Apache on the Web Server and mount it to NFS server’s export for logs

`sudo mount -t nfs -o rw,nosuid <NFS-Server-Private-IP-Address>:/mnt/logs /var/log/httpd`

Edit the /etc/fstab to make it permanently incase of reboot

`sudo vi /etc/fstab`

![Mounting](./Images/etc-web.png)

`sudo mount -a`

`sudo systemctl dameon-reload`

### Fork the tooling source code from the darey.io on github.

Install git first

`sudo yum install git -y`

`git clone https://github.com/darey-io/tooling.git`

Then change the directory to the tooling directory that was downloaded

`cd tooling`

Copy the content of the /html into /var/www/html

`sudo cp -R html/. /var/www/html`

Then go to the Public Address of the Webserver on the Browser

![MLanding Page](./Images/login.php.png)

### Connect the Website configuration to the Databaser server

`sudo /var/www/html/functions.php`

Edit the Private Ip address, the database name, user and password

![Database Connection](./Images/connect.png)
### Then Install Mysql client on the Webserver

`sudo yum install mysql`

`cd tooling`

Then run the follwowing script

```
mysql -h <databse-private-ip> -u webaccess -p tooling < tooling-db.sql
```

Then go into the browser and log in with the information already on the database as a user

![Last Page](./Images/propitix.png)


