# Ubuntu Linux Server Configuration: Udacity Full Stack Developer Project 6

This is a linux web server configuration project  -the 6th project in the Udacity Full Stack Nanodegree.

The goal is to get a basic Ubuntu instance from a cloud provider (I used Amazon Lightsail), secure it, then install the software required to host a Python Flask web application and Postgresql DB.

# Notes for Project Reviewer

*IP Address and SSH Port:*
- IP: 18.188.238.113
- SSH Port: 2200

*URL for hosted web app:*
- TODO

*Software Installed:*
- `apt-get install finger`

*Third Part Resources Used to Complete Project:*
- Add private key to SSH agent for remote login:
  - https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/
- Disable remote login with root user:
  - https://askubuntu.com/questions/27559/how-do-i-disable-remote-ssh-login-as-root-from-a-server

# Detailed Setup Steps

- Create Amazon Lightsail Ubuntu Instance
  - IP: 18.188.238.113
  - DNS: 18.188.238.113.xip.io
- Get and add SSH private key to ssh-agent
  - https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/
  - Start the ssh-agent in the background: `eval $(ssh-agent -s)`
  - Add private key to the ssh-agent: `ssh-add <path-to-private-key>`
  - Now we can SSH from our own SSH client to the instance
- Update server
  - `sudo apt-get update`
  - `sudo apt-get upgrade`
- Change SSH port to 2200
  - Update Amazon Lightsail Firewall, allow TCP on port 2200.
  - `sudo vi /etc/ssh/sshd_config` --- change `port 22` to `port 2200`.
  - `sudo service sshd restart`
  - Use `ssh -p 2200 user@server` to connect
  - Update Amazon Lightsail Firewall, deny TCP on port 22.
- Update Uncomplicated Firewall (UFW) to allow incoming connections
  - Allow SSH (2200), HTTP (80), NTP (123)
  - `sudo ufw status`
  - `sudo ufw default deny incoming`
  - `sudo ufw default allow outgoing`
  - `sudo ufw allow 2200/tcp`
  - `sudo ufw allow ntp`
  - `sudo ufw allow www`
  - `sudo ufw show added`
  - `sudo ufw enable`
  - `sudo ufw status` - Should display:
```
        To                         Action      From
        --                         ------      ----
        2200/tcp                   ALLOW       Anywhere
        123                        ALLOW       Anywhere
        80/tcp                     ALLOW       Anywhere
        2200/tcp (v6)              ALLOW       Anywhere (v6)
        123 (v6)                   ALLOW       Anywhere (v6)
        80/tcp (v6)                ALLOW       Anywhere (v6)
```
- Setup Users & Accesses to Server
  - Install Finger
    - `sudo apt-get install finger`
  - Disallow remote root login via SSH
    - `sudo vi /etc/ssh/sshd_config` --- add line `PermitRootLogin no`
	- `sudo service ssh restart`
  - Setup user `grader`
    - `sudo adduser grader` --- pw `grader`
    - Give sudo access to `grader`
      - `sudo cp /etc/sudoers.d/90-cloud-init-users /etc/sudoers.d/grader`
      - Update `/etc/sudoers.d/grader` to specify user `grader`
    - Setup SSH login
	  - Create SSH keypair
        - `ssh-keygen` --- passphrase `grader`
      - Install public key
	    - Switch user to `grader` and go to home directory
		- `mkdir .ssh`
		- `touch .ssh/authorized_keys`
		- Copy text from public key created in previous step to `authorized_keys`
		- `chmod 700 .ssh`
		- `chmod 644 .ssh/authorized_keys`
	  - Setup private key for SSH access via grader from local machine
	    - Repeat "Get and add SSH private key to ssh-agent" from above
  - At this point, only users `ubuntu` and `grader` should be able to log in remotely (only using their private SSH key, no passwords)
