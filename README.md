
# Udacity FSND Linux Server Configuration Project
## Project Description
This project takes a baseline installation of a Linux server and prepares it to host the Vehicle Catalog Project previously completed in the course, which was deployed locally with Vagrant. 
This server is configured to be secured against a number of attack vectors using Ubuntu's Uncomplicated Firewall(UFW). 

The Vehicle Catalog Project is also modified to use Postgres, which is configured on the server as well.

Finally, the server is configured to host the Vehicle Catalog Flask application publicly using Apache.

## Server details
The vehicle catalog project is hosted with [Amazon Lightsail](https://aws.amazon.com/lightsail/) on an Ubuntu 16.04 server.
The application URL is `http://52.14.196.229.xip.io/catalog` 
Please reference the [Vehicle Catalog Project](https://github.com/kchg/vehicle-catalog-project) for source code and descriptions for the application.

Note - Amazon Lightsail is a paid subscription-based service, so this URL will most likely not be available after the nanodegree completion. However, the steps outlined can be recreated on any Ubuntu instance, with changes to the host address. 
## Configuration Walkthrough

### Update Packages
```bash
sudo apt update
sudo apt upgrade
```
### Change SSH port to 2200
```bash
vi /etc/ssh/sshd_config
# change line from Port 22 to Port 2200
# restart ssh service
sudo service ssh restart
```

### Configure Firewall
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow www
sudo ufw allow ntp
sudo ufw enable
```

### Set up `grader` user
```bash
# create user without a password
sudo adduser grader --disabled-password
# give sudo permissions
sudo usermod -aG sudo grader
```
#### Set up SSH keys for `grader` user
```bash
# Generate ssh key pair on local machine
ssh-keygen
# On server, in grader home directory:
mkdir .ssh
chmod 700 .ssh
touch .ssh/authorized_keys
chmod 600 .ssh/authorized_keys
# Copy locally created public key to authorized_keys file
```
### Disable root login
```bash
sudo vim /etc/ssh/sshd_config
# set PermitRootLogin no
```

### Change timezone to UTC
Confirm timezone is set to UTC with `date` utility.
Otherwise, use:
```bash
sudo timedatectl set-timezone UTC
```

### Install dependencies
```bash
sudo apt install apache2 libapache2-mod-wsgi
sudo apt install postgresql
sudo apt install python-pip python-psycopg2 python-dev
sudo apt install git
```
### Ensure remote connections are disabled in Postgres
```bash
sudo vim /etc/postgresql/9.5/main/pg_hba.conf
# Check that only local connections are enabled
```
### Set up Postgres
```bash
sudo -u postgres psql

# Create catalog db with catalog user
sudo -u postgres createdb -O catalog catalog
```
### Pull code to Apache
```bash
cd /var/www
# Configure git, either through user/pass or ssh keys
git clone https://github.com/kchg/vehicle-catalog-project.git
```
### Set up Python virtual environment and requirements
```bash
cd /var/www/vehicle-catalog-project/deploy
pip install virtualenv
virtualen venv

# activate the virtualenv - 'deactivate' to exit
source venv/bin/activate

# install requirements
pip install -r requirements.txt
python database_setup.py
python addvehicles.py
```

### Create the WGSI Script
This is located at `/var/www/vehicle-catalog-project/deploy/`
```python
# app.wgsi
#!/usr/bin/python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/vehicle-catalog-project/")
from deploy.project import app as application
application.secret_key = 'super secret key'
```

### Configure Apache2
This is located at `/etc/apache2/sites-available/`. This virtual host configuration allows Apache to correctly handle domain requests.
```
# item-catalog.conf
<VirtualHost *:80>
        ServerName 52.14.196.229.xip.io
        ServerAdmin ubuntu@52.14.196.229
        WSGIDaemonProcess item-catalog python-home=/var/www/vehicle-catalog-project/deploy/venv
        WSGIProcessGroup item-catalog

        #WSGIApplicationGroup %{GLOBAL}

        WSGIScriptAlias / /var/www/vehicle-catalog-project/deploy/app.wsgi

        <Directory /var/www/vehicle-catalog-project/deploy>
                Order allow,deny
                Allow from all
                #Require all granted
        </Directory>

        # HTML, CSS
        Alias /static /var/www/vehicle-catalog-project/deploy/static

        <Directory /var/www/vehicle-catalog-project/deploy/static/>
                Order allow,deny
                Allow from all
                #Require all granted
        </Directory>


        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
#### Enable the virtual host configuration file
```bash
sudo a2ensite item-catalog.conf
```

### Set up Google OAuth Redirects
Modify OAuth settings under your client_secret using the Google Developers Console (https://console.developers.google.com/apis/credentials/)

Ensure that the http://52.14.196.229.xip.io URL is listed under 'Authorized Javascript Origins'

Add "Authorized redirect URIs":
 - http://52.14.196.229.xip.io/
 -  http://52.14.196.229.xip.io/catalog
 -  http://52.14.196.229.xip.io/gconnect

### Start/Restart the Apache Service
```bash
sudo service apache2 reload
```

## Resources
I used [this](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps) DigitalOcean tutorial as a guideline on the server set up. 
I referenced [this](https://modwsgi.readthedocs.io/en/develop/user-guides/virtual-environments.html) mod_wsgi document to set up the virtual environment.
