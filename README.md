# Udacity FSWD Project 6 - Server Configuration

<!-- MarkdownTOC autolink="true" -->

- [Synopsis](#synopsis)
- [My VPS Configuartions](#my-vps-configuartions)
    - [Public IP & HostNames](#public-ip--hostnames)
- [Create a VPS](#create-a-vps)
    - [Options](#options)
    - [Review](#review)
    - [Configuration](#configuration)
    - [Contracts](#contracts)
    - [Billing](#billing)
    - [Login & Passwords](#login--passwords)
        - [OVH account](#ovh-account)
        - [VPS](#vps)
- [Setup your VPS](#setup-your-vps)
    - [Initial Login & setup](#initial-login--setup)
        - [SSH Port](#ssh-port)
        - [Change root password](#change-root-password)
        - [Create a user with sudo privileges](#create-a-user-with-sudo-privileges)
        - [Remove the SSH root login](#remove-the-ssh-root-login)
    - [System Update && Upgrade](#system-update--upgrade)
        - [yum update](#yum-update)
        - [Install Kernel-ml](#install-kernel-ml)
        - [Inline with Upstream Stable repository](#inline-with-upstream-stable-repository)
    - [Install additional packages](#install-additional-packages)
        - [Git](#git)
        - [Server Configuartions](#server-configuartions)
        - [Postgresql](#postgresql)
        - [Python 3.X](#python-3x)
        - [virtualenv](#virtualenv)
        - [Apache 2.4.33](#apache-2433)
        - [Let's encrypt Certbot](#lets-encrypt-certbot)
        - [WSGI python3.6](#wsgi-python36)
    - [Advanced Server Setup](#advanced-server-setup)
        - [Prompt](#prompt)
        - [Timezone UTC](#timezone-utc)
        - [Chrony NTP Server](#chrony-ntp-server)
        - [Testing Chrony NTP Server](#testing-chrony-ntp-server)
        - [Secure SSH](#secure-ssh)
        - [Firewall configuration](#firewall-configuration)
        - [Start Apache service](#start-apache-service)
        - [Configure Apache for Catalog site](#configure-apache-for-catalog-site)
        - [Let's Encrypt Catalog certification](#lets-encrypt-catalog-certification)
        - [Reload the httpd server](#reload-the-httpd-server)
        - [SSL Cetificate test](#ssl-cetificate-test)
        - [Postgresql setup](#postgresql-setup)
        - [Catalog Website setup](#catalog-website-setup)
        - [Populating the database](#populating-the-database)
        - [Reload and Restart the Server](#reload-and-restart-the-server)
- [References](#references)

<!-- /MarkdownTOC -->


## Synopsis
Reading through the project requirements AWS Lightsail was recommended, as they provide a free trial.
I made an account and created an instance and after going through the inital setup of an Ubuntu VPS the following problems made me decide to choose another VPS provider:

* First I noticed the lack of options of Operating Systems. 
* Instead of using the default ssh key which was created by AWS, I wanted to used the most secure ed25519 and several faults were given that my chosen encryption was not supported.
* Other OS, i.e. RedHat and CentOS uses SELinux which uses MAC (Mandatory Access Control) ontop of DAC (Descretionary Access Control).
* **Cronyd** CentOS uses crony as standard, it is more reliable, faster, more accurate than ntpd. It also doesn't need any open ports to synchronise with online timeservers.

For These reasons I decided to create a server on [OVH](https://www.ovh.co.uk) and host our previous projects on the VPS.  
OVH has several choices for your VPS, in this instance we will use the basic [VPS SSD 1](https://www.ovh.co.uk/vps/vps-ssd.xml)  

We will use the latest available packages and standards including:

* http2 : HTTP 2.0 - faster, simpler, more secure
* Apache2.4.33 : latest package supporting http2
* WSGI : supporting Python3.6
* Cronyd : faster, more accurate, less memory & cpu intensive Network Time Protocol
* Python3.6 : python2.x is ending its livetime in less than 24 months
* LetsEncrypt : Free and secure HTTPS

## My VPS Configuartions
### Public IP & HostNames

* General hostname: [http://vps551706.ovh.net/](http://vps551706.ovh.net)
* Catalog hostname: [http://catalog.binops.co.uk/](http://catalog.binops.co.uk)
* Your VPS's IPv4 address is: [51.38.83.98](https://catalog.binops.co.uk)
* Your VPS's IPv6 address is: [2001:41d0:0801:2000:0000:0000:0000:0d39](https://catalog.binops.co.uk)

## Create a VPS
After following the above link press "redeem" and you'll be directed to the order page. When in London use the following settings or change according to your location.
<img src="./img/order-page.png" alt="Order Page" width="1138" height="640">
### Options
Choose the following options:

* VPS 2018 SSD 1
* Western Europe > London (UK)
* CentOS
* Centos 7 64bits
* English

and press "continue"

### Review
Choose your preferred plan, monthly, yearly, etc.

and press a gain "continue"

### Configuration
You will need to provide Responsibilities information for:

* Adminitstrator
* Billing
* Technician

Again press "continue"

### Contracts
Read and accept the T&C's and press "continue"

### Billing
Review your invoice and choose a payment option.

### Login & Passwords
#### OVH account
Go to [OVH](https://www.ovh.co.uk/) and login with your OVH account credentials, make sure you enable the (SMS) 2 factor authentication, to secure your account.

#### VPS
You will have received an email with the IP4 IPV6 address of your VPS together with a root login password.

Make sure you change the password upon login.

## Setup your VPS
### Initial Login & setup
Go to a terminal and ssh to the VPS
```bash
ssh root@51.38.83.98
```
Enter your password given in an email send by OVH.

#### SSH Port
To prevent brute force attacks on the default ssh port enter the following commands.
Make a backup of the sshd_config file
```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```
open the sshd_config file with vi and remove the **#** from te port line and change it to **2200**

Enable the newly created port in SELinux
```bash
sudo semanage port -a -t ssh_port_t -p tcp 2200
```

Add the port 2200 to the fierwall
```bash
sudo firewall-cmd --permanent --zone=public --add-port=2200/tcp
sudo firewall-cmd --reload
```
Reload the **sshd.service**
```bash
sudo systemctl restart sshd.service
```
Verify SSH is now running on port 2200
```bash
ss -tnlp | grep ssh
```
Ths shoud output 2 lines showing ssh with ports 2200

To make sure open a second terminal and login on port 2200
```bash
ssh root@51.38.83.98 -p 2200
```

#### Change root password
Since the original password was send by email, there is always a chance it was compromised in transit. So change the root password.
```bash
passwd root
```

#### Create a user with sudo privileges
To prevent from mistakes being made as a su, A normal user should be created with sudo privileges
```bash
useradd grader
passwd grader
usermod -a -G wheel grader
```

#### Remove the SSH root login
To finish off our basic server ssh configuration, we want to prevent ssh root logins.  
First we will log out and log back in with our new user grader.
```bash
ssh -p 2200 grader@51.38.83.98
sudo vi /etc/ssh/sshd_config
```
go to the line with `#PermitRootLogin yes` and change it to `PermitRootLogin no`

### System Update && Upgrade
#### yum update
Make sure that the basic system has all the latest packages
```bash
sudo yum update
```

#### Install Kernel-ml
Kernels are regularly being update and security holes are plugged, so it is imperative to use the latest kernel updates.  
The following 2 lines will install the elrepo repository and its GPG key.
```bash
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm 
```
Install the latest kernel
```bash
sudo yum --enablerepo=elrepo-kernel install kernel-ml
```
Open and edit the file **/etc/default/grub** and set `GRUB_DEFAULT=0` above `GRUB_DISABLE_SUBMENU`  
Then run the following command to recreate the kernel configuartion.  
And reboot the VPS
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo systemctl reboot
```

#### Inline with Upstream Stable repository
IUS community brings latest software versions to RHEL-based systems.
```bash
sudo yum install https://centos7.iuscommunity.org/ius-release.rpm
```

### Install additional packages
#### Git
Install Git to retrieve additional packages
```bash
sudo yum install git
```

#### Server Configuartions
We will clone several helper files onto the server with git.  
Go to your home directory and clone the Server Config.
```bash
git clone https://github.com/tordne/Udacity-server-config.git
```

#### Postgresql
We'll install postgresql to use for our item-catalog project
```bash
sudo yum install postgresql postgresql-server
```

#### Python 3.X
Install the latest python version.
```bash
sudo yum install python36u python36u-pip
```

#### virtualenv
Python venv does not contain the `activate_this.py` file, hence we prefer to use virtualenv
```bash
sudo python3.6 -m pip install virtualenv
```

#### Apache 2.4.33
We will install the latest apache which will support http2
```bash
sudo yum install httpd24u python2-certbot-apache
```

#### Let's encrypt Certbot
When using logins and databases it should be standard for sites to use https.  
Certbot gives free SSL certificates which can be renewed every 3 months.
```bash
sudo yum install httpd24u-mod_ssl
```

#### WSGI python3.6
Install the latest mod_wsgi
```bash
sudo yum install python36u-mod_wsgi
```

### Advanced Server Setup
#### Prompt
CentOS default prompt is bland and makes it difficult to view information.
```bash
cd Udacity-server-config
sudo cp prompt.sh /etc/profile.d/
```
`exit` the VPS and log back in with grader, now you will have a fancy prompt.

#### Timezone UTC
The timezone is set in `/etc/localtime` which is a symlink to one of the time zones in `/usr/share/zoneinfo/`.  
To change it we'll use the following command.
```bash
sudo timedatectl set-timezone UTC
```

#### Chrony NTP Server
To open the ntp server open the `/etc/chrony.conf` file and add the following line under `#Allow NTP client access from local network`
```bash
allow
```
Adding this line without a subnet will allow clients from all addresses to use the ntp.  
Now restart the `Chronyd service`
```bash
sudo systemctl restart chronyd
```
#### Testing Chrony NTP Server
Check that chronyc is synchronising with external NTP servers.  
The number of sources should be not 0 and higher than 2, to properly function.
```bash
sudo chronyc sources
```
Then you can also test the server side is working properly.
Open 2 terminals and 1 wireshark.  
In wireshark start capturing on your default network interface and set the filter to `udp.port == 123`  
Terminal 1 should be ssh in `grader@XX.XX.XX.XX` and run the command
```bash
sudo tcpdump -A -vv -i eth0 'port 123'
```
Terminal 2 should be local and execute the following command.
```bash
sudo nmap -sU -p 123 XX.XX.XX.XX
```
This will send a request to your NTP server, and you should see the following:

* tcpdump & Wireshark have captured your local request and server response
* nmap will show that port 123/UDP is open
* <img src="./img/ntp-test.png" alt="Wireshark, nmap NTP test" width="1138" height="640">

#### Secure SSH
To protect your VPS against brute force password attacks we'll use passwordless ssh login.  
**On your client do the following:**  
Create a strong ed25519 key
```bash
ssh-keygen -o -a 100 -t ed25519
```
create a config file `~/.ssh/config` with permission **600** and enter:
```bash
Host grader-udacity
  Hostname 51.38.83.98
  Port 2200
  User grader
  PreferredAuthentications publickey
  IdentityFile ~/.ssh/id_ed25519
```
**On your open terminal do the following**  
Copy the contents of public key `id_ed25519.pub` in the file `~/.ssh/authorized_keys`  
Then modify `/etc/ssh/sshd_config` with sudo and change  
`PasswordAuthentication yes` to `PasswordAuthentication no`  
and  
`PubkeyAuthentication no` to `PubkeyAuthentication yes`  
then restart the sshd service
```bash
sudo systemctl sshd.service
```

#### Firewall configuration
Firewalld is active and has multiple zones, of which by default only public is active with several ports.  
We will need to close port 22 and open others i.e. http, https, ntp
```bash
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --permanent --zone=public --add-service=ntp
sudo firewall-cmd --permanent --zone=public --remove-service=ssh
sudo firewall-cmd --reload
```

#### Start Apache service
After installing Apache2.4.33 the service has not yet been enabled. So
```bash
sudo systemctl enable httpd.service
sudo systemctl restart httpd.service
```

#### Configure Apache for Catalog site
To create multiple virtual hosts we need to prepare apache
```bash
sudo mkdir /etc/httpd/sites-{enabled,available,variables}
```
Create a file called `catalog.conf` and copy the following inside
```html
WSGISocketPrefix /var/run/wsgi

<VirtualHost *:80>
  ServerName catalog.XXXXX.net
      Include /etc/httpd/sites-variables/catalog-vars.conf

    WSGIDaemonProcess catalog.binops.co.uk user=cataloguser group=cataloguser threads=5 home=/var/www/catalog/item-catalog/ python-path=/var/www/catalog/item-catalog:/var/www/catalog/env/lib/python3.6/site-packages/
    WSGIScriptAlias / /var/www/catalog/item-catalog/catalog.wsgi

    <Directory /var/www/catalog>
        WSGIProcessGroup catalog.binops.co.uk
        WSGIApplicationGroup %{GLOBAL}
        WSGIScriptReloading On
    Require all granted
    </Directory>

    LogLevel info
    ErrorLog /var/log/httpd/catalog.log
    CustomLog /var/log/httpd/catalog-requests.log combined

    RewriteEngine on
    RewriteCond %{SERVER_NAME} =catalog.binops.co.uk
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
</VirtualHost>


<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName catalog.binops.co.uk

    Include /etc/httpd/sites-variables/catalog-vars.conf

    Protocols h2 http/1.1
    Include /etc/letsencrypt/options-ssl-apache.conf

    WSGIScriptAlias / /var/www/catalog/item-catalog/catalog.wsgi
  
    <Directory /var/www/catalog>
         WSGIProcessGroup catalog.binops.co.uk
         WSGIApplicationGroup %{GLOBAL}
         WSGIScriptReloading On
         Require all granted
    </Directory>

    LogLevel info
    ErrorLog /var/log/httpd/catalog443.log
    CustomLog /var/log/httpd/catalog443-requests.log combined
    SSLCertificateFile /etc/letsencrypt/live/catalog.binops.co.uk/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/catalog.binops.co.uk/privkey.pem
    Header always set Strict-Transport-Security "max-age=31536000"
    Header always set Content-Security-Policy upgrade-insecure-requests
</VirtualHost>
</IfModule>
```
Make a link to sites-available
```bash
sudo ln -s /etc/httpd/sites-available/catalog.conf catalog.conf
```
Let apache search through the enabled sites by appending the following code to `/etc/httpd/conf/httpd.conf`
```bash
# Load virtual host config files in the "/etc/httpd/sites-enabled" directory, if any.
Include sites-enabled/*.conf
```
Create the `/etc/httpd/sites-variables/catalog-vars.conf` with all the site specific variables.
```html
SetEnv FLASK_APP /var/www/catalog/item-catalog/catalog.py
SetEnv FLASK_DEBUG 0
SetEnv FLASK_CONFIG /var/www/catalog/item-catalog/config.py
SetEnv FLASK_SETTINGS config.DevelopmentConfig
SetEnv CLIENT_SECRET_FILE /var/www/catalog/item-catalog/client_secret.json
SetEnv DATABASE_URL postgresql://catalog_admin:CXXXXXXXX@localhost:5432/catalogdb
```
Secure the site with a strong CipherSuite and SSL Protocol.  
Append the following code to `/etc/httpd/conf.d/ssl.conf` behind `</VirtualHost>`
```bash
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

SSLCipherSuite ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:!aNULL:!eNULL:!EXPORT:!DES:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!IDEA:!ECDSA

SSLHonorCipherOrder On
Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
# Requires Apache >= 2.4
SSLCompression off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
# Requires Apache >= 2.4.11
SSLSessionTickets Off
```
To solve some http2 problems we need to use the MPM event module.  
Open wit Vi `/etc/httpd/conf.modules.d` and comment out all the `LoadModule ...` except for the `LoadModule mpm_event_module modules/mod_mpm_event.so`

#### Let's Encrypt Catalog certification
Let's encrypt will create SSL certificates for our site with a expiration of 3 months.  
When asked to redirect all http to https enter number `2`
```bash
sudo certbot --apache --non-interactive --agree-tos --redirect --hsts --uir --rsa-key-size 4096 --domain catalog.XXXX.net
```
This will have created another configuration file called `/etc/httpd/site-available/catalog-le-ssl.conf`  
Add the following line after `DocumentRoot /var/www/catalog` to ensure it will try to use http2
```bash
Protocols h2 http/1.1
```
Open with VI the `/etc/letsencrypt/options-ssl-apache.conf` and comment out the line `SSLProtocol` and `SSLCipherSuite` and add the following code below them.
```bash
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1

SSLCipherSuite ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:!aNULL:!eNULL:!EXPORT:!DES:!LOW:!3DES:!MD5:!EXP:!PSK:!DSS:!RC4:!SEED:!IDEA:!ECDSA
```
#### Reload the httpd server
Now you'll need to reload the httpd server with either
```bash
sudo systemctl restart httpd.service
```
or
```bash
sudo apachectl restart
```
#### SSL Cetificate test
Go to [SSLlabs](https://www.ssllabs.com/ssltest/) and enter your webaddress and check that your site is secure. it shoud receive at least an A+

#### Postgresql setup
First we will need to enable the postgresql server
```bash
sudo systemctl enable postgresql
sudo postgresql-setup initdb
```
Enable password authentication in `/var/lib/pgsql/data/pg_hba.conf`.  
Open the file with vi and search for the lines
```bash
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
# IPv6 local connections:
host    all             all             ::1/128                 ident
```
Change it to
```bash
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
```
Start or Restart the postgresql service
```bash
sudo systemctl restart postgresql
```
Set password for postgres user.
```bash
sudo passwd postgres
```
then login as postgres user using su.
```bash
su - postgres
```
Enter postgresql and create a new Database user and grant privileges.
```bash
psql
postgres=# CREATE DATABASE catalogdb;
CREATE DATABASE
postgres=# CREATE USER catalog_admin WITH PASSWORD 'C1t1l0gFr01t';
CREATE ROLE
postgres=# GRANT ALL PRIVILEGES ON DATABASE catalogdb TO catalog_admin;
GRANT
postgres=# \q
```
Then exit the postgres bash session. with `exit`.  

#### Catalog Website setup
First we add a user with its home directory under /var/www
```bash
sudo useradd -m -d /var/www/catalog -c "Catalog User" cataloguser
sudo passwd cataloguser
```
Login as cataloguser
```bash
su - cataloguser
```
Go to the home directory and create a virtual environment and activate
```bash
cd
virtualenv env
source env/bin/activate
```
Clone the git repo
```bash
git clone https://github.com/tordne/item-catalog.git
```
Enter the item-catalog directory and install all the necessary packages
```bash
cd item-catalog
pip install --upgrade pip
pip install -r requirements.txt
```
Create the `/var/www/catalog/item-catalog/catalog.wsgi` file and enter the following code
```python
#!/usr/bin/env python

import os

''' Start the virtual environment '''
activate_this = '/var/www/catalog/env/bin/activate_this.py'
with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

''' Start the wsgi application '''
def application(req_environ, start_response):
   
   os.environ['FLASK_APP'] = req_environ['FLASK_APP']
   os.environ['FLASK_DEBUG'] = req_environ['FLASK_DEBUG']
   os.environ['FLASK_CONFIG'] = req_environ['FLASK_CONFIG']
   os.environ['FLASK_SETTINGS'] = req_environ['FLASK_SETTINGS']
   os.environ['CLIENT_SECRET_FILE'] = req_environ['CLIENT_SECRET_FILE']
   os.environ['DATABASE_URL'] = req_environ['DATABASE_URL']

   from project.server import app as _application

   return _application(req_environ, start_response)
```
Go to google developers console and download your client_secret.json and place it under `/var/www/catalog/item-catalog/client_secrent.json`  
Copy the `config-template.py` to config.py and change the following lines with the appropriate content
```python
SECRET_KEY = 'very_secret_key'
CLIENT_ID = '101XXXXXXX-l3hXXXXXXXX8e.apps.googleusercontent.com '
CLIENT_SECRET = 'tVXXXXXXXXJ7 '
CLIENT_SECRET_FILE = '/vagrant/client_secret.json'
SQLALCHEMY_DATABASE_URI = 'postgresql://vagrant:vagrant@localhost/catalog'
```
#### Populating the database
We have set up a database which has not yet been populated so execute the following command to temporarily set the env variables and execute `lotsOfCategoriesAndItems.py`
```bash
export FLASK_APP=/var/www/catalog/item-catalog/catalog.py
export FLASK_DEBUG=1
export FLASK_CONFIG=/var/www/catalog/item-catalog/config.py
export FLASK_SETTINGS=config.DevelopmentConfig
export CLIENT_SECRET_FILE=/var/www/catalog/item-catalog/client_secret.json
python lotsOfCategoriesAndItems.py
```
#### Reload and Restart the Server
To reload Apache and WSGI execute the following 2 codes.
```bash
touch /var/www/catalog/item-catalog/catallog.wsgi
sudo systemctl restart httpd
```

## References
[Change default port CentOS 7](https://www.liberiangeek.net/2014/11/change-openssh-port-centos-7/)  
[Secure your SSH](https://www.hugeserver.com/kb/secure-ssh-on-centos-7/)  
[CentOS 7 update kernel](https://www.tecmint.com/install-upgrade-kernel-version-in-centos-7/)  
[Encryption hardening](https://bettercrypto.org/static/applied-crypto-hardening.pdf)
