# VPS Setup

If you want to set up a production ready VPS, there are a few steps you should take.

This document goes through the list of steps that I personally take.


## 1. Create a New User with Sudo Permissions
```
# Log in as root
ssh root@your-server-ip

# Create a new user
adduser newuser

# Add the user to the sudo group
usermod -aG sudo newuser

# Test the new user
su - newuser
sudo apt update
```


## 2. Set Up SSH Key Authentication
```
# On your local machine, generate an SSH key pair if you don’t already have one
ssh-keygen -t ed25519 -C "your_email@example.com"

# Copy the SSH key to the new user on the server
ssh-copy-id -i ~/.ssh/id_ed25519.pub newuser@your-server-ip

# Test key-based login
ssh root@your-server-ip # ensure root is disabled
ssh newuser@your-server-ip
```

## 3. Edit DNS Records 
```
# Remove DNS A + CNAME (I'm using namecheap as domain provider)
https://ap.www.namecheap.com/Domains/DomainControlPanel/hamburgersvshotdogs.com/advancedns

# in HOST RECORDS section
remove all records
add:
  Type-A Record
  Host-@
  Value-<vps-ip-address>
  TTL-5min 

# Check if DNS Record has propagated and DNS resolves to new IP
# inside of ssh:
nslookup <your-domain-name.com>

# (after step 4. and hardening) test SSH with domain name
ssh -i ~/.ssh/id_hostinger_vps_01 newuser@<your-domain-name.com>
```

## 4. Harden SSH

```
# Open SSH configuration file
sudo nano /etc/ssh/sshd_config

# Modify the following in the file:
# PasswordAuthentication no  # Disable password based auth
# UsePAM no
# PermitRootLogin no # Disable root login

# HOSTINGER ssh auth 
  sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf

  # Modify the following in the file:
  # PasswordAuthentication no  

# Restart SSH service
sudo systemctl restart ssh

# Test SSH with new settings before logging out
ssh newuser@your-server-ip 
(above did not work for me, maybe cause of custom .ssh pub name, so I have to enter the following:)
ssh -i ~/.ssh/id_hostinger_vps_01 newuser@your-server-ip 

```

## 4.1. Setup Local Machine Docker SSH Config + SSH-Agent 
## for ssh ease of use
```
# Create ssh config
nano ~/.ssh/config

# Add config (example below:)
Host burgvps
    HostName burgvshotdogs.com
    User newuser
    IdentityFile ~/.ssh/id_vps_01
    IdentitiesOnly yes

# use ssh agent to start the day
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_hostinger_vps_01
```

## 5. Set Up a Firewall (UFW)
```
# Install UFW if not already installed
sudo apt install ufw

# Allow necessary ports
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH    # SSH
sudo ufw allow 80/tcp     # HTTP
sudo ufw allow 443/tcp    # HTTPS

sudo ufw show added

# Enable UFW
sudo ufw enable

# Check UFW status 
sudo ufw status verbose # verbose prints out default incoming + outgoing

# Continue to reverse-proxy integration
(nginx or recommended: traefik)
https://youtu.be/F-9KWQByeU0?si=MQk0ztNjmDaWYVki&t=985
```
## 6. Install Docker + Docker Compose on VPS
```
https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

# on step 2 reduce bloat by:
sudo apt-get install docker-ce docker-ce-cli containerd.io -----#reduce bloat by not running= 'docker-buildx-plugin docker-compose-plugin'

# Confirm Proper Install
run 'sudo docker run hello-world'. A good way to confirm Docker is setup properly!
run 'sudo docker ps'

# If its not running, set system control cmd:
sudo systemctl enable docker

# Add user name to docker group to prevent sudo requirement: 
# https://docs.docker.com/engine/install/linux-postinstall/
sudo usermod -aG docker $USER
# relogin to have permissions take affect
exit
docker ps

```

## 7. Setup Docker Image using Docker Stack + Compose
```
https://www.youtube.com/watch?v=fuZoxuBiL9o

Docker Stack allows you to deploy your Docker Compose onto a node that has Docker Swarm enabled

This workflow supports
- Blue/Green deployment
- Rolling releases
- Secure Secrets
- Rollbacks
- Remote Deploys
- CI/CD using GitActions
- Clustering


# Deploy webapp remotely
# Change Docker Host to that of our VPS using Docker Contexts
# https://docs.docker.com/engine/manage-resources/contexts/
# https://youtu.be/fuZoxuBiL9o?si=gKUty9Toy_dBVmp3&t=543

docker context create <name-of-webapp-site> --docker <define-endpoint-to-that-of-ssh-endpoint>
ex:
docker context create my-webapp-site --docker "host=ssh://newuser@mywebapp.com"


```




## (Optional) Install and Configure Fail2Ban

```
# Install Fail2Ban
sudo apt install fail2ban

# Create a local configuration file
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# Edit Fail2Ban configuration for SSH
sudo nano /etc/fail2ban/jail.local
# Ensure the following lines are set:
# [sshd]
# enabled = true
# port = 22 # Change this if you've modified your SSH port.
# maxretry = 5
# bantime = 3600

# Restart Fail2Ban service
sudo systemctl restart fail2ban

# Check Fail2Ban status
sudo fail2ban-client status
sudo fail2ban-client status sshd
```
