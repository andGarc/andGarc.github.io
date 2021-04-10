Learn how to create backend environment for web mapping and GIS purposes. 
This backend environnement is simple and low-cost to build. It is build using Geoserver and PostgreSQL, more specifically PostGIS. 

## Objectives
1. Create a web geospatial stack consisting of PostgreSQL and Geoserver
2. Learn to utilize Amazon Web Services(AWS)

---

## Create a AWS Account

Go to AWS's [website](https://aws.amazon.com) and create a new user account.
A Free-tier usage account is enough to get the main gist behind this process. 

---

## Set Up AWS

Go to your AWS Management Console.
Set your region to N.Virginia. You can set this on the top right corner of the page.
Click on the drop-down next to your user name and select US East (N. Virgina).

Next we will set up an EC2 instance. 

An EC2 instance is a virtual server in the AWS cloud. EC2 stand for Elastic Compute Cloud. 

From the Services drop-down menu on the top left select EC2.
A new window will open. Click launch instance then select Ubuntu Server 18.04.
![](/assets/images/geostack/launch_ec2.png)
![](/assets/images/geostack/select_ubuntu.png)

You will be directed to Step 2 of the set up. 
t2.micro will be selected by default. 
Leave t2.micro as is and click Next. 
![](/assets/images/geostack/t2.png)

Leave the Step 3 as is and click Next: Add Storage. 

In Step 4 increase the storage to 30gb and click next to continue. 
![](/assets/images/geostack/storage.png)

In Step 5, add tags that oyu think will be beneficial.
Click Next: Configure Security Group.

Now, in Step 6, we will configure the security group. This is were we set up the type of connections from and to our new instance. 
Fow now, we will only grant access to our computer. 
Under Source, select My IP and add a description. 
![](/assets/images/geostack/security_group.png)

You can always edit these rules later if you need to grant access via SSH to other users. 

Click Review and Launch. 

You will be redirected to a summary window. If everything looks good, then click the Launch button. 

You will now be prompted to create a a key pair. This key pair is required to access the instance via SSH. 
Select Create a new key pair, name it, and download it.

Launch the Instance.
This might take a couple minutes.
To see if your instance is ready to go, click on view instance or select EC2 from the Services drop-down menu. The state of the instance will say 'running' and we are ready to go. 

Next we need to allocate and associate an elastic ip to the instance. This is a requirement for being able to point a domain to our instance (in case you want to). 

In your EC2 instance dashboard, under Network and Security, click Elastic IPs. 
![](/assets/images/geostack/elastic_ip.png)

Click Allocate Elastic IP Address.
Leave the defaults and click Allocate.

The new ip will be listed on the screen. 
Make sure the ip is selected then under Actions click Associate Elastic IP address. 
![](/assets/images/geostack/associate.png)

Select your instance from the instance dropdown, leave everything else as is and click Associate. 

Your EC2 instance is not ready and can be accessed via SSH. We know install [Geoserver](http://geoserver.org) and [PostGIS](https://postgis.net) on it.

---

## Set Up Geoserver

Open a terminal window.
Using your `.pem` key ssh into your EC2 instance. 
You will need to get the Public IPv4 DNS address from the instance. This is located at the bottom of the page under Details in the AWS Instance dashboard.  

Connect to your instance.
```console
ssh -i /pathtokeypair/keypair.pem ubuntu@ec2-yourDNSname.amazonaws.com
```

Note: if you get `private key file error` you will need to troubleshoot this. Refer to [this](https://99robots.com/how-to-fix-permission-error-ssh-amazon-ec2-instance/) article on how to do this.  

Now that we are connected, we can install Geoserver. 

First, let's update our package list:
```console
sudo apt-get update
```

Now, let's install openjdk java 8. 
```console
sudo apt-get install openjdk-8-jre
```

Install Apache Tomcat 8, a prerequisite for Geoserver. 
```console
sudo apt-get install tomcat8
```
Create a folder called `Downloads` and download Geoserver here.
```console
mkdir Downloads
cd Downloads

sudo wget http://sourceforge.net/projects/geoserver/files/GeoServer/2.19.0/geoserver-2.19.0-war.zip
```

You can use the `ls` command to see the newly downloaded zip file.

Install unzip and unzip the file.
```console
sudo apt-get install unzip
unzip geoserver-2.19.0-war.zip
```
Use the `ls` command to see the unzipped files. You should see a `geoserver.war` file.

Move the `geoserver.war` file into `tomcat8/webapps`.
```console
sudo mv geoserver.war /var/lib/tomcat8/webapps/

# check that file was moved correctly
ls /var/lib/tomcat8/webapps/
```

Restart the tomcat8 service.
```console
sudo service tomcat8 restart
```

Geoserver has been installed. Before you can access it you need to edit the security group for your instance and grant access to port 8080 (Geoserver's default port).

Go to the EC2 dashboard and follow the previous steps to set up a security group. All you need to do, is edit the Inbound rules by adding a new Custom TCP rule with your ip as a source and 8080 as the port range. 
Note: you can make this public and accessible to anyone with the url by picking Anywhere as the source.

After you  setup the new inbound rule, Geoserver can be access via the `http://youDNSofEC2:8080/geoserver` url.
Go to the url and check it out!

Geoserver's default username and password are:
```
Username: admin
Password: geoserver
```

Log in and change the password. 

---

## Set Up PostgreSQL

Now, we will install a PostgreSQL database with PostGIS.
PostGIS is a spatial database extender for PostgreSQL.

First, use the cd command and go back to the home directory. 
```console
cd ~
```

Update the packages again, install PostgreSQL and install PostGIS.
```console
sudo apt update 
sudo apt install postgresql postgresql-contrib
sudo apt-get install postgis*
```

Now we need to change the password of the default user postgres.
```console
# switch to pg console
sudo -u postgres psql
postgres=#\password​
# enter new password
```

Install the PostGIS extension. 
```console
# Create the extension
CREATE EXTENSION postgis;
```

Cool, now we have set up PostgreSQL and enabled PostGIS. 
Use the `\q` command to exit the pg console. 

In order to to be able to access the database through a local [pgAdmin](https://www.pgadmin.org) instance on our computers we need to edit the pg configuration files.  
PostgreSQL currently only listens to the EC2 local host, so we need to edit the configuration files. 

First, edit the `pg_hba.conf` for remote access for clients. 
```console
# Use vim to edit pg_hba.conf
sudo vim /etc/postgresql/10/main/pg_hba.conf

# Near bottom of file after local rules, add rule (allows remote access)
# press I and add the following line
host    all             all             0.0.0.0/0               md5

# save, press the ESC key then :wq
```

Edit the `postgresql.conf` file to allow for external access.
```console
sudo vim /etc/postgresql/10/main/postgresql.conf

# Change line 59 to listen to external requests:
listen_address='*'

# save
```

Lastly, restart PostgreSQL.
```console
# Restart postgres
sudo /etc/init.d/postgresql restart
```

Again, before you can access postgres you need to edit the security group's inbound rules for your instance and grant access to port 5432 (PostgreSQL's default port).

You can now access your PostgreSQL database on your own machine. 

---

We have created an AWS instance and installed Geoserver and PostgreSQL. We have completed the geospatial stack!
This set up is relatively free so our resources (CPU and memory) are limited. I suggest you use a more powerful instance to increase performance. Beyond that, there are other methods we can use to improve the performance of this set up, but are out of the scope of this walk through.


