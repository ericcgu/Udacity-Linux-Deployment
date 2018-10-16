<a href="https://www.udacity.com/">
  <img src="https://s3-us-west-1.amazonaws.com/udacity-content/rebrand/svg/logo.min.svg" width="300" alt="Udacity logo svg">
</a>

# Udacity-Linux-Deployment

## Provisioning and Securing a Server

### Cloud Partner

For this project I chose Digital Ocean because only a single instance is required for this project.

After manually collecting the optimal steps, I plan to design a Terraform script to automate a fault-tolerant cluster of instances on AWS/Azure/Google.

### Environment: Server and User Setup

- I followed the [DigitalOcean Initial Server Setup with Ubuntu 16.04 tutorial](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04).

- The first step is to SSH as root:

  ```shell
  ssh root@165.227.182.114
  ```

- At this point, root can log in with a password

- After logging in as root, I created a `grader` user for the Udacity grader with sudo rights.

  ```shell
  adduser grader
  usermod -aG sudo grader
  cut -d: -f1 /etc/passwd                                       #Confirm users
  ```

- Generate a SSH key pair

  ```shell
  touch ~/.ssh/config
  ssh-keygen -t rsa -b 4096 -C "eric.changning.gu@gmail.com"
  ls -al ~/.ssh                                                 #Confirm new SSH Key
  ssh-add -K ~/.ssh/udacity_grader A                            #Add SSH private key to the ssh-agent
  ```
  
- Copy SSH Key to Server using `ssh-copy-id`.

  ```shell
  ssh-copy-id -i ~/.ssh/udacity_grader grader@165.227.182.114
  ssh grader@165.227.182.114 #test SSH authentication
  ```
 
- Update, upgrade and clean packages, then restart.

  ```shell
  sudo apt update
  sudo apt upgrade
  sudo apt autoremove
  sudo reboot
  ```

- Set time zone to UTC:

  ```shell
  sudo dpkg-reconfigure tzdata
  ```

## Security

- Configure and Enable Firewall

  ```shell
  sudo ufw app list
  sudo ufw allow 2200
  sudo ufw allow 2200/tcp
  sudo ufw allow 80/tcp
  sudo ufw allow 123/udp
  sudo ufw enable
  ```

  ```text
  $ sudo ufw status
  Status: active

  To                         Action      From
  --                         ------      ----
  OpenSSH                    ALLOW       Anywhere
  2200/tcp                   ALLOW       Anywhere
  80/tcp                     ALLOW       Anywhere
  123/udp                    ALLOW       Anywhere
  2200                       ALLOW       Anywhere
  OpenSSH (v6)               ALLOW       Anywhere (v6)
  2200/tcp (v6)              ALLOW       Anywhere (v6)
  80/tcp (v6)                ALLOW       Anywhere (v6)
  123/udp (v6)               ALLOW       Anywhere (v6)
  2200 (v6)                  ALLOW       Anywhere (v6)
  ```

- Change SSH port from 22 to 2200

    ```shell
    sudo nano /etc/ssh/sshd_config
    ```

    ```text
    # What ports, IPs and protocols we listen for
    Port 2200
    # Change to no to disable tunnelled clear text passwords
    PasswordAuthentication no
    PermitRootLogin no
    ```

  - Save and quit with ctrl+x.
  - Restart SSH

    ```shell
    sudo systemctl reload sshd
    service ssh restart
    ```

  - Exit and log back in, this time specifying port 2200.

    ```shell
    ssh grader@165.227.182.114 -p 2200 -i ~/.ssh/udacity_grader
    ```
    
- The *~/.ssh/config* file on your local machine can also be configured for easier login. This file is on my local machine, so I was able to to open it with vscode.

  ```shell
  $ code ~/.ssh/config
  ```

  ```text
  Host udacity_grader
    Hostname 165.227.182.114
    User grader
    Port 2200
    PubKeyAuthentication yes
    IdentityFile ~/.ssh/udacity_grader

  Host *
    AddKeysToAgent yes
    UseKeychain yes
    IdentityFile ~/.ssh/id_rsa
  ```

- If the config file is set up as above, log in with `ssh udacity_grader`.

## Application Functionality

My Item Catalog submission involved a single docker container containing Python Flask + SQLite.
(https://github.com/ericcgu/Udacity-Flask-Item-Catalog)

In order to expand the functionality to add:

* `Web Server` 
* `WSGI`
* `PostgreSQL`

I needed to add containers. 

I followed hundreds of breadcrumbs across the internet to get the right mix and structure:

## IP Address

165.227.182.114

## URL

http://165.227.182.114.xip.io/

## Summary of Software

* `Docker` 
* `Python`
* `Flask` 
* `SQLAlchemy`
* `Nginx`
* `Gunicorn`
* `PosgreSQL`

## Third Party Resources

(https://realpython.com/dockerizing-flask-with-compose-and-machine-from-localhost-to-the-cloud/)
(https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-16-04)
(http://www.patricksoftwareblog.com/how-to-use-docker-and-docker-compose-to-create-a-flask-application/)


[(Back to top)](#top)
