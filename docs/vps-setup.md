# VPS Setup

"If you want to set up a production ready VPS, there are a few steps you should take.
This document goes through the list of steps that I personally take." - dreams-of-code

https://www.youtube.com/watch?v=F-9KWQByeU0

https://www.youtube.com/watch?v=fuZoxuBiL9o

https://youtu.be/gqseP_wTZsk?si=ibuWXj_CYWSLb10d

## 1. Create a New User with Sudo Permissions
```
# Log in as root
ssh root@your-server-ip

# Create a new user
adduser <new-user-name>

# Add the user to the sudo group
usermod -aG sudo <new-user-name>

# Test the new user
su - <new-user-name>
sudo apt update
```


## 2. Set Up SSH Key Authentication
```
# On your local machine (in ubuntu terminal), generate an SSH key pair if you don’t already have one
# cd to .ssh 
# ~/.ssh
ssh-keygen -t ed25519 -C "<ssh_id_name>"
# file in which to save?
<ssh_id_name>
# enter passphrase?
# enter + save inside keepass

# Copy the public SSH key to the new user on the server
ssh-copy-id -i ~/.ssh/<ssh_id_name>.pub <username>@your-server-ip

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

## 6.1 Git Hub Container Registry + GH Personal Access Token setup (GHCR + PAT)
```
https://youtu.be/gqseP_wTZsk?si=TY-p0vGc51AkqSY2

# Guide to Pushing Docker Images to Github to be easily accessible
# Build local image (in vscode project powershell terminal)
# Runs *Dockerfile* instructions
# # docker build --target production -t <image-name> . #-- run only production build
# # docker build --target development -t <image-name> . #-- run only development build

docker build -t <image-name> .

# once built, confirm it exists
docker images

# Generate Personal Access Token
https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-with-a-personal-access-token-classic
https://youtu.be/gqseP_wTZsk?si=wKbIKz0TOczsiiF0&t=196

# Set PAT Token as a control panel User Env Var, access by running in linux:
cmd.exe /C "echo %<ENV-VAR>%"

# docker login using env var
echo "<PAT>" | docker login ghcr.io -u <github-username> --password-stdin
# should show 'login succeeded"

# Now get ready to push to GHCR
docker tag <image-name> ghcr.io/<github-username>/<image-name>:latest
docker push ghcr.io/<github-username>/<image-name>:latest

# to link this Docker image directly to a repository, add the following to Dockerfile
# inside === PRODUCTION STAGE ====
LABEL org.opencontainers.image.source=https://github.com/<github-username>/<repo-name>

```

## 7. Remote Deployment -- Setup Docker Context 
```
# Deploy webapp remotely
# Change Docker Host to that of our VPS using Docker Contexts
# https://docs.docker.com/engine/manage-resources/contexts/
# https://youtu.be/fuZoxuBiL9o?si=gKUty9Toy_dBVmp3&t=543

docker context create <name-of-webapp-site> --docker <define-endpoint-to-that-of-ssh-endpoint>
# eg:
docker context create my-webapp-site --docker "host=ssh://newuser@mywebapp.com"
docker context create deploy-ham-vs-hot --docker "ssh://burgdeployvps" #-- see step 4.1 for using config! (nano ~/.ssh/config)

# Now can use docker context use, and any commands then will take place in the Docker Instance inside our VPS!
# allowing us to configure remotely
docker context use my-webapp-site
docker context show #--show name of current context
```

### 7.1 Remote Deployment -- Setup Docker Stack
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

# Enable Docker Swarm on VPS
docker swarm init
# upon running cmd, will receive token allowing to connect other VPS instances to this machine (in order to form a Docker Swarm Cluster)
# can easily obtain token

# With swarm mode enabled, can now deploy our App using Docker Stack Deploy Cmd, passing path to docker compose .yml and name of stack
# Ensure you are logged into GHCR to access PAT Token! See previous step 6. # docker login using env var
https://youtu.be/fuZoxuBiL9o?si=BmLaBjpVd-Llnrnh&t=678
docker stack deploy -c ./docker-stack.yaml <name-of-stack>
# for private GHCR package
docker stack deploy --with-registry-auth -c ./docker-stack.yaml <name-of-stack>

# to view all containers and confirm stack is good
docker ps -a
docker service ls

# To Delete/Clear up docker stack that you dont want run:
docker stack ls
docker stack rm <docker-stack-name>
docker system prune -a --volumes #-- removes any leftover resources


```

## 8. Docker Secret setup
```
# Create new secret
docker secret create <name-of-secret>

# load in through file OR through stdin
docker secret create db-password -./passwords.txt
# using stdin with printf
printf 'mysecretpassword' | docker secret create db-password -
# view secrets by running
docker secret ls
docker secret inspect <secret-ID>

# store secret in another safe spot
https://youtu.be/fuZoxuBiL9o?si=nc5AebGJCx_n5q3e&t=904

# In docker compose.yaml + docker stack.yaml, define secret as external
secrets:
  db-password:
    external: true

```

## 9. Automated Deployments
```
# Basically rebuilding a whole new VPS User but with only access to GitHub Docker Deployment commands in SSH
# https://youtu.be/fuZoxuBiL9o?si=otffAoUjvbXUUBdM&t=1213

# create new user in vps
adduser <new-user-name>

# Test the new user
su - <new-user-name>

# Add user name to docker group to prevent sudo requirement: 
# https://docs.docker.com/engine/install/linux-postinstall/
sudo usermod -aG docker <new-user-name>
groups <new-user-name> # to check current groups
# relogin to have permissions take affect
exit
docker ps

# Generate SSH Key
# See Step 2.

# after generating ssh key 
# copy .pub file to clipboard on local machine in ubuntu terminal
wl-copy < <ssh_id_name>.pub

# build .ssh dir in vps
su - deploy
mkdir .ssh
echo '<pasted-wl-copy-ssh-pub-key>' > .ssh/authorized_keys

# Login on the VPS as deploy user
# Make sure you SSH in (or use Docker context shell) and log in to ghcr.io with a token. 
# See Step 6.1
# This allows us to grab private GHCR keys
echo "<PAT>" | docker login ghcr.io -u <github-username> --password-stdin

# docker context create workaround for VPS connection 

# see steps 4.1 to build config!
# Host <deploy-<webapp-name>>
## User deploy (or whatever you named for secured deploy vps account)
## 4.1. Setup Local Machine Docker SSH Config + SSH-Agent 
## for ssh ease of use

# see steps 7 for new context create!
# docker context create my-webapp-site --docker "host=ssh://<deploy-<webapp-name>>"
# eg: docker context inspect:
        "Endpoints": {
            "docker": {
                "Host": "ssh://<deploy-<webapp-name>>",
                "SkipTLSVerify": false
            }

# restrict deploy user commands usable on ssh
nano .ssh/authorized_keys
# add following text before the ssh key
command="docker system dial-stdio" #-- this restricts user to only able to perform 'docker stack deploy' command when using ssh with this key

# can try running docker stack deploy now. example:
# make sure to change docker context to using new deploy user for endpoint! 
blake@LAPTOP-U8PV5H9S:/mnt/c/users/blake/.../dreams-of-code-vps-guide-02$ docker stack deploy -c docker-stack.yml <name-of-stack>


# with that, ready to add this private key to github repo
https://youtu.be/fuZoxuBiL9o?si=JG_fRPCKpa0ImBLf&t=1461
# Navigate to github repository's settings > secrets & variables > actions
# select 'New Repository Secret'
$ add private ssh key value that lives inside local .ssh folder
# Set name as SAME value as defined in github action!
# e.g. 'DEPLOY_MY_SSH_PRIVATE_KEY'
# look for e.g. <deploy_webapp_id>, will contain following text:
## -----BEGIN OPENSSH PRIVATE KEY-----
# paste into Secret value in Github

# time to create folder .github\workflows\
# and child file: <deploy-using-github-actions>.yml
```

## 10. Ensure DB is non-ephemeral
```
# ensure correct docker context and correct container is selected
docker service ls #-- (look for _db)

# check db data and what it is mounted to
# mounts dict inside
docker inspect <container-name-of-db>
#eg: docker inspect ham-test-08_db
##### "Source": "ham-test-08_db-data",

docker run --rm -it -v <Source>:/data alpine sh
#eg: docker run --rm -it -v ham-test-08_db-data:/data alpine sh
/ # ls -l /data
exit

docker ps #-- (look for postgres-latest container id)
docker exec -it <db-container-ID> psql -U postgres -d app
#eg: docker exec -it c28f177780 psql -U postgres -d app

# you should now be in the PostgresSQL CLI!
\dt #--  list tables
\d users #--  describe a table (assuming a table named 'users' exists)
SELECT * FROM users; #-- will list all data in 'users' table, may need to \q if you cant leave)
\l #--  list databases
\q #--  quit

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
