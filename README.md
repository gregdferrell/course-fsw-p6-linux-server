# Ubuntu Linux Server Configuration: Udacity Nanodegree - Full Stack Web Developer Project 6

This is a linux web server configuration project -the 6th project in the Udacity Nanodegree: Full Stack Web Developer.

The goal is to get a basic Ubuntu instance from a cloud provider (I used Amazon Lightsail), secure it, then install the software required to host a Python Flask web application and Postgresql DB.

## Demo

View a live demo at [storytime.gregdferrell.com](http://storytime.gregdferrell.com).

#### Software Installed:
```
finger
apache2
postgresql
git
python3-pip
libapache2-mod-wsgi-py3
```

#### Third Party Resources Used to Complete Project:
- Add private key to SSH agent for remote login:
  - https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/
- Disable remote login with root user:
  - https://askubuntu.com/questions/27559/how-do-i-disable-remote-ssh-login-as-root-from-a-server
- Postgresql reference
  - https://www.postgresql.org/docs/9.5/static/
- Using mod_wsgi w Flask
  - http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/
- File permissions for user uploads
  - https://stackoverflow.com/questions/21797372/django-errno-13-permission-denied-var-www-media-animals-user-uploads
- Getting Google OAuth login working (JavaScript origins) with Amazon Lightsail instance
  - https://discussions.udacity.com/t/solved-configuring-linux-google-oauth-invalid-request/376259/16

# Detailed Setup Steps

#### Create Amazon Lightsail Ubuntu Instance
- Ubuntu 16.04 LTS
- python3 --version => 3.5.2
- DNS: 52.14.121.23.xip.io

#### To login w SSH, add SSH Private Key to ssh-agent
```
# Get SSH private Key from Amazon Lightsail Account Page

# On local machine, start the ssh-agent in the background:
eval $(ssh-agent -s)

# Add private key to the ssh-agent:
ssh-add <path-to-private-key>

# Now we can SSH from our own SSH client to the instance
```

#### Update Server Software Packages
```
sudo apt-get update
sudo apt-get upgrade
```

#### Change SSH port to 2200
```
# Update Amazon Lightsail Firewall, allow TCP on port 2200.
# Change `port 22` to `port 2200` in the following file:
sudo vi /etc/ssh/sshd_config`

# Restart SSH
sudo service sshd restart

# Now, we can use the following to connect
ssh -p 2200 user@server

# Now, update Amazon Lightsail Firewall to deny TCP on port 22.
```

#### Update Uncomplicated Firewall (UFW) to Allow Incoming Connections
```
# Allow SSH (2200), HTTP (80), NTP (123)
sudo ufw status
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp
sudo ufw allow ntp
sudo ufw allow www
sudo ufw show added
sudo ufw enable

sudo ufw status - Should display:

        To                         Action      From
        --                         ------      ----
        2200/tcp                   ALLOW       Anywhere
        123                        ALLOW       Anywhere
        80/tcp                     ALLOW       Anywhere
        2200/tcp (v6)              ALLOW       Anywhere (v6)
        123 (v6)                   ALLOW       Anywhere (v6)
        80/tcp (v6)                ALLOW       Anywhere (v6)
```

#### Setup Users & Accesses to Server
```
# Install Finger
sudo apt-get install finger

# Disallow remote root login via SSH
# Add line `PermitRootLogin no` to the following
sudo vi /etc/ssh/sshd_config`
sudo service ssh restart`

# Setup user `grader` with pw `grader`
sudo adduser grader

# Give sudo access to `grader`
sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader
# Then update `/etc/sudoers.d/grader` to specify user `grader`

# Setup SSH login
# Create SSH keypair for `grader`, passphrase `grader`
ssh-keygen`

# Install public key on server
# Switch user to `grader` and go to home directory
su - grader
cd ~
mkdir .ssh
touch .ssh/authorized_keys

# Copy text from public key created in previous step to `authorized_keys`
# Update permissions on authorized keys
chmod 700 .ssh
chmod 644 .ssh/authorized_keys

# Setup private key for SSH access via grader from local machine
# Repeat "Get and add SSH private key to ssh-agent" from above
# At this point, only users `ubuntu` and `grader` should be able to log in remotely (only using their private SSH key, no passwords)
```

#### Install Apache2, Python 3 WSGI, PIP3, Postgresql
```
sudo apt-get install apache2

# Visit your IP and see the default Apache2 page
```

#### Configure Postgresql
```
sudo apt-get install postgresql
```

###### Setup User: `postgres`
```
# Test connection
psql -U postgres -W
> psql: FATAL:  Peer authentication failed for user "postgres"

# Run the psql command from the postgres user account and set the postgres user's password to `postgres`:
sudo -u postgres psql postgres
\password postgres

# Update config file: /etc/postgresql/9.5/main/pg_hba.conf
# Set `peer` to `md5` for local connections from all users (2 lines)
sudo vi /etc/postgresql/9.5/main/pg_hba.conf

# Restart postgres
sudo service postgresql restart

# Test connection again (this time it should work)
psql -U postgres -W
```

###### Setup DB & User: `catalog`
```
# Add Linux User: catalog
sudo adduser catalog

# Add Postgresql DB: catalog
sudo -u postgres createdb catalog

# Add Postgres User: catalog
createuser -U postgres --interactive catalog
# Answer "N" to the 3 questions, minimizing this user's privileges
# User: 'catalog', Pass: 'catalog'

# From psql, Set User catalog as Owner of DB catalog:
psql -U postgres
ALTER DATABASE catalog OWNER TO catalog;
ALTER USER catalog WITH PASSWORD 'catalog';
```

#### Setup Python Web App

###### Install Git and Download Source
```
# Git may already be installed
sudo apt-get install git

cd /var/www
sudo git clone https://github.com/gregdferrell/fsw-p4-story-time.git
```

###### Setup App Config
```
Follow instructions for application configuration on the app GitHub page: https://github.com/gregdferrell/fsw-p4-story-time

# Facebook App
Add "http://<ip-address/domain-name>" to "Valid OAuth Redirect URIs"

# Google App
# https://console.developers.google.com/apis/credentials?project=<your-project-name>
Add "http://<domain>" to "Authorized JavaScript origins"
- Note: You cannot just use an IP address. When using Amazon Lightsail, you need to figure out what the DNS findable URL is. Mine was: http://ec2-52-14-121-23.us-east-2.compute.amazonaws.com
Add "http://<domain>/oauth2callback" to "Authorized redirect URIs"
```

###### Create DB Schema and Add Test Data
```
# Create schema
cd /var/www/fsw-p4-story-time/db
psql -U catalog -d catalog -a -f create_schema.sql

# Add test data
sudo apt-get install python3-pip
sudo pip3 install -r /var/www/fsw-p4-story-time/requirements.txt
sudo python3 /var/www/fsw-p4-story-time/db/create_test_data.py
```

###### Set File Permissions for User Uploads of Images
```
# Permit www-data user to read/write/execute in /var/www/ for image uploads
sudo groupadd varwwwusers
sudo adduser www-data varwwwusers
sudo chgrp -R varwwwusers /var/www/
sudo chmod -R 770 /var/www/

# Add user ubuntu to this group so we can traverse the directory
# Note: User will have to log in again to obtain privileges associated with this group
sudo usermod -a -G varwwwusers ubuntu
```

###### Setup Apache2 for Python Web App
```
sudo apt-get install libapache2-mod-wsgi-py3

# Update the Apache2 config file
sudo vi /etc/apache2/sites-enabled/000-default.conf
# Add the following line to the VirtualHost element:
WSGIScriptAlias / /var/www/fsw-p4-story-time/storytime/app_prod.wsgi

# Restart Apache2
sudo apache2ctl restart
```

# PROFIT!
