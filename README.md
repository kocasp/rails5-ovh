# Guide for setting up rails 5 with action cable on OVH VPS server

## Setup ssh login for deploy

- ssh root@xxx.xxx.xxx.xxx providing a password from email
- create non-root user `useradd -d /home/deploy -m deploy` and set a password for him `passwd deploy`
- run `visudo` and add a line below
```
deploy ALL=(ALL) ALL
```
- run `vi /etc/ssh/sshd_config` and change following lines:
```
Port 4321
PermitRootLogin no
...
AllowUsers deploy
```
- restart ssh
```
service ssh restart
```
- set bash for deploy user
```
chsh -s /bin/bash deploy
```
- log in as deploy via `ssh deploy@xxx.xxx.xxx.xxx -p 4321` and run:
```
mkdir ~/.ssh
```
- log out from server
- being on local, add id_rsa.pub to authorized_keys on the server
```
cat ~/.ssh/id_rsa.pub | ssh deploy@xxx.xxx.xxx.xxx -p 4321 'cat - >> ~/.ssh/authorized_keys'
```
- log in by `ssh deploy@xxx.xxx.xxx.xxx -p 4321` and change authorized_keys permissions by:
```
sudo chmod 600 ~/.ssh/authorized_keys && chmod 700 ~/.ssh/
```
- ssh login with `deploy@xxx.xxx.xxx.xxx -p 4321` is now enabled and secured

## Setup server environment
 - prepare your system
```
$ sudo apt-get update
$ sudo apt-get install curl -y
```
- generate ssh key:
```
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```
- install ruby using RVM: (in case it fails, run commands mentioned in the error message and try again)
```
$ \curl -L https://get.rvm.io | bash -s stable --ruby
```
- run:
```
source /home/deploy/.rvm/scripts/rvm
```
- install ruby 2.2.2 and set it as default
```
$ rvm install 2.2.2
$ rvm --default use 2.2.2
```
- install bundler:
```
gem install bundler
```
- install Node.js
```
sudo apt-get install -y nodejs
```

## Add server key to the repository access

- on server, check and copy id_rsa.pub with `cat ~/.ssh/id_rsa.pub | pbcopy` and add its content to repository (bitbucket/github) deployment keys

## Install and setup postgresql db on server:

```
$ sudo apt-get install -y postgresql postgresql-contrib
$ sudo apt-get install -y libpq-dev
$ sudo apt-get clean
```
- on server: create user for app (replace sampleuser with your username and password with your password)
```
$ sudo -u postgres psql -c "CREATE USER sampleuser WITH PASSWORD 'password' CREATEDB;"
$ sudo -u postgres psql -c "CREATE DATABASE app_production;"
$ sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE app_production to sampleuser;"
$ sudo -u postgres psql -c "ALTER USER sampleuser WITH SUPERUSER;"
```
- create database.yml in shared directory
```
$ mkdir -p ~/apps/literki/shared/pids
$ mkdir -p ~/apps/literki/shared/log
$ mkdir -p ~/apps/literki/releases
$ mkdir -p ~/apps/literki/shared/config
$ cd ~/apps/literki/shared/config
$ vi database.yml
```
paste following content in vi editor:
```
production:
  adapter:  postgresql
  host:     localhost
  encoding: unicode
  database: app_production
  username: sampleuser
  password: password
```
- create application.yml in shared directory. Being still in shared/config folder:
```
$ touch application.yml
```
## Install nginx server
- run
```
sudo apt-get install -y nginx
```
- next run
```
sudo vi /etc/nginx/sites-enabled/default
```
erase all fily by pressing ESC then type :1,$d and ENTER
and paste the following:
```
#this should be placed under /etc/nginx/sites-enabled/default
#remember about changing xxx.xxx.xxx.xxx to IP address and updating app paths

upstream unicorn {
  server unix:/tmp/unicorn.spui.sock fail_timeout=0;
}

server {
  listen 80 default deferred;
  server_name xxx.xxx.xxx.xxx;
  root /home/deploy/apps/literki/current/public;
  error_log  /etc/nginx/nginx_error.log  warn;

  location ^~ /assets/ {
    gzip_static on;
    expires max;
    add_header Cache-Control public;
  }

  location /cable {
     proxy_pass http://unicorn/cable;
     proxy_http_version 1.1;
     proxy_set_header Upgrade websocket;
     proxy_set_header Connection Upgrade;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   }

  location ~ ^/(robots.txt|sitemap.xml.gz)/ {
    root /home/deploy/apps/literki/current/public;
  }

  try_files $uri/index.html $uri @unicorn;
  location @unicorn {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unicorn;
  }

  error_page 500 502 503 504 /500.html;
  client_max_body_size 4G;
  keepalive_timeout 10;
}


```
- run puma from current directory:
```
bundle exec puma -e production -b unix:///home/deploy/apps/letters/shared/tmp/sockets/letters-puma.sock --daemon
```
- install redis server and run as daemon
```
$ sudo apt-get install redis-server
$ redis-server --daemonize yes
```
- restart nginx with:
```
sudo service nginx restart
```
- install git
```
sudo apt-get install git
```
