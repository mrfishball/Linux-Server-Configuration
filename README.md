# Linux-Server-Configuration

## Description

A baseline installation of a Linux distribution on a virtual machine and prepare it to host your web applications, to include installing updates, securing it from a number of attack vectors and installing/configuring web and database servers.

The application meant to be deployed is the **O2 app**, previously developed (https://github.com/mrfishball/O2).

## Useful info

IP address: 35.163.62.121.

Accessible SSH port: 2200.

Application URL: [http://ec2-35-163-62-121.us-west-2.compute.amazonaws.com/](http://ec2-35-163-62-121.us-west-2.compute.amazonaws.com/).

## Step by step walkthrough

### 1 - Create a new user named *grader* and grant this user sudo permissions.

1. Log into the remote VM as *root* user through ssh: `$ ssh root@35.163.62.121`.
2. Add a new user called *grader*: `$ sudo adduser grader`.
3. Create a new file under the suoders directory: `$ sudo nano /etc/sudoers.d/grader`. Fill that newly created file with the following line of text: "grader ALL=(ALL:ALL) ALL", then save it.
4. In order to prevent the "sudo: unable to resolve host" error, edit the hosts:
	1. `$ sudo nano /etc/hosts`.
	2. Add the host: `127.0.1.1 ip-10-20-27-45`.

### 2 - Update all currently installed packages

1. `$ sudo apt-get update`.
2. `$ sudo apt-get upgrade`.
3. Install *finger*, a utility software to check users' status: `$ apt-get install finger`.

### 3 - Configure the local timezone to UTC

1. Open time configuration dialog and set it to UTC with: `$ sudo dpkg-reconfigure tzdata`.
2. Install *ntp daemon ntpd* for a better synchronization of the server's time over the network connection: `$ sudo apt-get install ntp`.

Source: [UbuntuTime](https://help.ubuntu.com/community/UbuntuTime).

### 4 - Create and configure the key-based authentication for *grader* user


1. Generate an encryption key **on your local machine** with: `$ ssh-keygen`.
2. Enter path with file name as `/home/vagrant/.ssh/grader`.
3. Copy the content of the grader.pub file with: `$ cat .ssh/grader.pub`.
4. Log into the remote VM as *root* user through ssh then switch to user *grader* with `$ su - grader` and create the following file: `$ touch /home/grader/.ssh/authorized_keys`.
5. Paste the content copied earlier from grader.pub in the authorized_keys file with `$ sudo nano .ssh/authorized_keys`. Then change some permissions:
	1. `$ sudo chmod 700 /home/grader/.ssh`.
	2. `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`.
	3. Finally change the owner from *root* to *grader*: `$ sudo chown -R grader:grader /home/grader/.ssh`.
6. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/udacity_key.rsa grader@35.163.62.121`.

### 5 - Enforce key-based authentication
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *PasswordAuthentication* line and edit it to *no*.
2. `$ sudo service ssh restart`.

### 6 - Change the SSH port from 22 to 2200
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *Port* line and edit it to *2200*.
2. `$ sudo service ssh restart`.
3. Now you are able to log into the remote VM through ssh with the following command: `$ ssh -i ~/.ssh/udacity_key.rsa -p 2200 grader@35.163.62.121`.

### 7 - Disable ssh login for *root* user
1. `$ sudo nano /etc/ssh/sshd_config`. Find the *PermitRootLogin* line and edit it to *no*.
2. `$ sudo service ssh restart`.

### 8 - Configure the Uncomplicated Firewall (UFW)

Project requirements need the server to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

1. `$ sudo ufw allow 2200/tcp`.
2. `$ sudo ufw allow 80/tcp`.
3. `$ sudo ufw allow 123/udp`.
4. `$ sudo ufw enable`.

### 9 - Configure cron scripts to automatically manage package updates

1. Install *unattended-upgrades* if not already installed: `$ sudo apt-get install unattended-upgrades`.
2. To enable it, do: `$ sudo dpkg-reconfigure --priority=low unattended-upgrades`.

### 10 - Install Apache, mod_wsgi

1. `$ sudo apt-get install apache2`.
2. Mod_wsgi is an Apache HTTP server mod that enables Apache to serve Flask applications. Install *mod_wsgi* with the following command: `$ sudo apt-get install libapache2-mod-wsgi python-dev`.
3. Enable *mod_wsgi*: `$ sudo a2enmod wsgi`.
3. `$ sudo service apache2 start`.

### 11 - Install Git

1. `$ sudo apt-get install git`.
2. Configure your username: `$ git config --global user.name <username>`.
3. Configure your email: `$ git config --global user.email <email>`.

### 12 - Clone the Catalog app from Github

1. `$ cd /var/www`. Then: `$ sudo mkdir O2`.
2. Change owner for the *O2* folder: `$ sudo chown -R grader:grader O2`.
3. Move inside that newly created folder: `$ cd O2` and clone the catalog repository from Github: `$ git clone https://github.com/mrfishball/O2.git O2`.
4. Make a *O2.wsgi* file to serve the application over the *mod_wsgi*. That file should look like this:

```python

#!/usr/bin/python
import sys
import os
import logging

os.environ['APP_MAIL_USER'] = 'hahahehe8787@gmail.com'
os.environ['APP_MAIL_PASS'] = 'password123'

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0,"/var/www/O2/O2")

from app import app as application
application.secret_key = 'uglymojo'

```

### 13 - Install virtual environment, Flask and the project's dependencies

1. Install *pip*, the tool for installing Python packages: `$ sudo apt-get install python-pip`.
2. If *virtualenv* is not installed, use *pip* to install it using the following command: `$ sudo pip install virtualenv`.
3. Move to the *O2* folder: `$ cd /var/www/O2/O2`. Then create a new virtual environment with the following command: `$ sudo virtualenv venv`.
4. Activate the virtual environment: `$ source venv/bin/activate`.
5. Change permissions to the virtual environment folder: `$ sudo chmod -R 777 venv`.
6. Install Flask: `$ pip install Flask`.
7. Install all the other project's dependencies: `$ pip install bleach httplib2 request oauth2client sqlalchemy python-psycopg2`. 

### 14 - Configure and enable a new virtual host

1. Create a virtual host conifg file: `$ sudo nano /etc/apache2/sites-available/O2.conf`.
2. Paste in the following lines of code:
```
<VirtualHost *:80>
    ServerName 35.163.62.121
    ServerAdmin admin@35.163.62.121
    WSGIScriptAlias / /var/www/O2/O2.wsgi
    <Directory /var/www/O2/O2/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/O2/O2/app/static
    <Directory /var/www/O2/O2/app/static/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/O2/O2/app/templates
    <Directory /var/www/O2/O2/app/templates/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

```
3. Enable the new virtual host: `$ sudo a2ensite catalog`.

### 15 - Install and configure PostgreSQL

1. Install some necessary Python packages for working with PostgreSQL: `$ sudo apt-get install libpq-dev python-dev`.
2. Install PostgreSQL: `$ sudo apt-get install postgresql postgresql-contrib`.
3. Postgres is automatically creating a new user during its installation, whose name is 'postgres'. That is a tusted user who can access the database software. So let's change the user with: `$ sudo su - postgres`, then connect to the database system with `$ psql`.
4. Create a new user called 'o2' with his password: `# CREATE USER o2 WITH PASSWORD 'sillypassword';`.
5. Give *o2* user the CREATEDB capability: `# ALTER USER o2 CREATEDB;`.
6. Create the 'catalog' database owned by *o2* user: `# CREATE DATABASE o2 WITH OWNER o2;`.
7. Connect to the database: `# \c o2`.
8. Revoke all rights: `# REVOKE ALL ON SCHEMA public FROM public;`.
9. Lock down the permissions to only let *o2* role create tables: `# GRANT ALL ON SCHEMA public TO o2;`.
10. Log out from PostgreSQL: `# \q`. Then return to the *grader* user: `$ exit`.
11. Inside the config.py of the Flask application(/var/www/O2/O2,) the database connection is now performed with: 
```python
SQLALCHEMY_DATABASE_URI = 'postgresql://o2:uglymojo@localhost/o2'
```
12. Setup the database with: `$ python /var/www/O2/O2/create_db.py`.
13. To prevent potential attacks from the outer world we double check that no remote connections to the database are allowed. Open the following file: `$ sudo nano /etc/postgresql/9.3/main/pg_hba.conf` and edit it, if necessary, to make it look like this: 
```
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```
### 16 - Restart Apache to launch the app
1. `$ sudo service apache2 restart`.
