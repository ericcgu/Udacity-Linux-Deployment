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

I added and orchestrated additional containers, keeping in line with the single responsibility principle, decoupling each piece of functionality.  

For example, it is possible to swap PostgreSQL with MySQL with a single line of code change.

(https://github.com/ericcgu/Flask-Gunicorn-Postgres-Docker)

```bash
.
├── db
├── web
├── nginx
├── postgres
├── .env
└── docker-compose.yml
```

![docker-application-architecture](https://user-images.githubusercontent.com/4943759/46992807-ea2b7f80-d0d9-11e8-8266-41b53e0aff04.png)


```dockerfile
FROM python:3.7.0-slim
LABEL maintainer "Eric Gu <eric.changning.gu@gmail.com>"
RUN apt-get update \
    && apt-get upgrade -y \                                                 # I updated the docker file to increase security
    && apt-get install -y \
    && apt-get -y install apt-utils gunicorn libpq-dev python3-dev \
    && apt-get autoremove -y \
    && apt-get clean all
ENV INSTALL_PATH /usr/app/static
RUN mkdir -p $INSTALL_PATH
WORKDIR $INSTALL_PATH
COPY requirements.txt .
RUN pip install --upgrade pip -r requirements.txt
COPY . .

CMD ["/web/entrypoint.sh"]
```
```yml
version: '3.7'

services:
  postgres:
    build: ./postgres
    restart: always
    volumes:
      - ./db:/var/lib/postgresql
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: 'master'
      POSTGRES_USER: 'postgres'

  web:
    restart: always
    build: ./web
    environment:
      ENV: "Production"
      PYTHONUNBUFFERED: 1
      PYTHONDONTWRITEBYTECODE: 1
      OAUTHLIB_RELAX_TOKEN_SCOPE: 1
    ports:
      - "8000:8000"
    expose:
      - 8000
    depends_on:
      - postgres
    links:
      - postgres:postgres
    volumes:
      - .:/usr/src/app/static
    env_file: .env
    command: gunicorn -w 2 -b :8000 itemcatalog:app

  nginx:
    build: ./nginx
    restart: always
    volumes:
      - /www/static
    ports:
      - "80:80"
    depends_on:
      - web

```
## Deployment:

To deploy this application configuration, there was four simple steps:

1. Create an personal access token
2. Create an instance of Docker Machine with personal access token

```bash
 docker-machine create \
-d digitalocean \
--digitalocean-access-token=ADD_YOUR_TOKEN_HERE \
production
```

3. Run `eval "$(docker-machine env production)"`

4. run `docker-compose up --build`

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
