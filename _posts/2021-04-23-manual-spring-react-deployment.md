---
layout: post
title: Manual deployment process for SpringFramework + NextJS on Ubuntu and Nginx platform
comments: true
tags: [spring-framework,java,nginx,ubuntu,deployment,nextjs]
---
### 0. Legend

- local machine - the machine you are using to access your server
- server - your ubuntu server

&nbsp;

### 1. Platform Setup

1. Install Linux OS to any cloud VM of your choice (DigitalOcean droplet?) and install Ubuntu 20 or later. Make sure to login as root (or a sudoer user)

2. Update ubuntu packages repo 

   ```
   apt-get update
   ```

3. Install Nginx webserver

   ```
   apt-get install nginx
   ```

4. Check your default firewall app list and allow necessary ports

   ```
   ufw app list
   ufw allow 22 //allows access request via SSH
   ```

5. For security, allow ssh access by password is not recommended. However, as initial setup, you may need to temporarily allow this to setup your SSH keys. 

   ```
   pico /etc/ssh/sshd_config
   ```

   Make sure that you convert the value of `PermitRootLogin` and `PasswordAuthentication` to `yes`

   ```
   PermitRootLogin yes
   PasswordAuthentication yes
   ```

6. You may now access remotely via SSH (w/ password authentication)

   ```
   ssh <username>@<ip_address>
   ```

&nbsp;

### 2. Setup Public/Private keys for Remote access via SSH without password (MacOS context)

1. On your Local Machine (MacOS), copy public key or generate one using `ssh-keygen -t rsa`

   ```
   ssh-keygen -t rsa //execute only if you don't have existing ssh keys
   
   cat ~/.ssh/id_rsa.pub //copy the public key displayed
   ```

2. On your server, go to your keys file and paste public key from local machine.

   ```
   pico ~/.ssh/authorized_keys //this is the file where your paste your public key.
   ```

   Save and exit editor after pasting. Then edit ssh config file.

   ```
    pico /etc/ssh/sshd_config
   ```

   Find and change back the `PasswordAuthentication` to "no" value.

   ```
   PasswordAuthentication no
   ```

   Save and exit editor then make sure to restart your SSH Service.

   ```
   service sshd restart
   ```

3. Now close your terminal and access back your ubuntu server via SSH from your local machine terminal. Notice that it will not ask for a password anymore.

   

&nbsp;

### 3. Installation of Java, Postgre, Redis, NodeJS/npm and nginx_ensite

1. Install Java with the following command:

   ```
   apt install openjdk-8-jre-headless
   ```

   After installation, test if java installation is succesful by typing:

   ```
   java -version //this should display your java version
   ```

2. Install PostgreSQL

   ```
   apt-get install postgresql postgresql-contrib
   ```

   Change default postgres password

   ```
   sudo -u postgres psql
   ```

   Then type: 

   ```
   ALTER USER postgres PASSWORD 'new_password';
   
   --To exit type:
   \q
   ```

   I suggest that you use a Postgre Manager (connect via SSH) to create database and manage tables.

   Recommended: pgAdmin, TablePlus, dBeaver etc.

3. Install Redis (Optional. Only if your application uses redis service)

   ```
   sudo apt install redis-server
   ```

   To check redis status, type: 

   ```
   systemctl status redis
   ```

4. Install NodeJS and NPM

   ```
   apt install nodejs
   ```

   Verify installation by typing: 

   ```
   node --version
   ```

   Verify also `npm` installation by typing: 

   ```
   npm --version
   ```

   If `npm` was not found, install it by typing 

   ```
   apt install npm
   ```

5. Install nginx_ensite. This application allows easy enabling and disabling of nginx config files.

   You may choose where you want to store nginx_ensite but let's assume you are on your root folder.

   ```
   git clone https://github.com/perusio/nginx_ensite.git
   
   cd nginx_ensite && make install 
   ```

   If you don't have `make` module on your server, install it by typing: 

   ```
   apt-get install make
   ```

   or if you want to install the whole essential libs/apps (which also contains the `make`), you may install 

   ```
   sudo apt-get install build-essential
   ```

&nbsp;

### 4. Setting up of your Java and Next app

1. Make a folder on where you want to store your Java app (.jar file) and your NextJS folder. For the sake of this tutorial, let's assume you place it on your `/home/project`

   Folder structure:

   ```
   /home/project/<app_name>.jar
   
   /home/project/<next_folder>
   ```

   Make sure that you have initiated a build already on your next folder. If not, then type:

   ```
   cd /home/project/<next_folder> && npm run build
   ```

2. Create a daemon/service to easily manage your application.

   Go to System folder: 

   ```
   cd /etc/systemd/system
   ```

   (This may not be best practice or may have more other practical ways but anyway....)

   Create 2 service files: 1 for your Java app and 1 for your Next app

   ```
   touch <java_app_name>.service
   
   touch <next_app_name>.service
   ```

   ------------------------

   > - At this point, let us assume that when you run java and next app, 
   >
   > it will run on port `8080` and port `3000`
   >
   > - Let us also assume that you have already setup a domain that points to each application. 
   >
   > Example: `srv.myapp.com` for Java and `myapp.com` for Frontend/Next app 

   Edit the newly created service files and paste the following content:

   ```
   /------- For <java_app_name>.service --------- /
   [Unit]
   Description=Java API Service
   After=syslog.target
   After=network.target[Service]
   
   [Service]
   ExecStart=/usr/bin/java -jar /home/project/<app_name>.jar
   Restart=always
   StandardOutput=syslog
   StandardError=syslog
   SyslogIdentifier=javaapp
   
   [Install]
   WantedBy=multi-user.target
   
   /--------- For <next_app_name>.service ----------/
   [Unit]
   Description=Next App Service
   After=syslog.target
   After=network.target[Service]
   
   [Service]
   ExecStart=/usr/bin/npm start --prefix /home/project/<next_folder>
   Restart=always
   StandardOutput=syslog
   StandardError=syslog
   SyslogIdentifier=nextapp
   
   [Install]
   WantedBy=multi-user.target
   ```

3. Run you application services:

   ```
   service <java_app_name> start // to start java
   
   service <next_app_name> start // to start next app
   ```

4. Next is you may need to configure nginx to make sure that when user from outside access your application via port `80` it will be redirected to their corresponding run ports. 

   To do this, you need to create 2 config files for nginx:

   ```
   cd /etc/nginx/sites-available
   ```

   There is already a default file called `default` that listens to port `80` ( you can see this by typing `ls`). However, we want to create a file that just contains enough config for our app. So for now, let us just disable `default` file and create our own 2 files `javaapp`and `nextapp`

   To do this, type:

   ```
   touch javaapp
   
   touch nextapp
   ```

   The commands above will create 2 empty files `javapp` and `nextapp`. You may verify this by typing `ls`.

   Next is to write your nginx config to the newly created files:

   ```
   /------ For javaapp file -----/
   server {
   				listen 80;
           listen [::]:80;
           
           root /var/www/html;
   
           index index.html index.htm index.nginx-debian.html;
   
           server_name srv.myapp.com;
   
           location / {
                   proxy_pass http://localhost:8080;
                   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                   proxy_set_header X-Forwarded-Proto $scheme;
                   proxy_set_header X-Forwarded-Port $server_port;
           }
   }
   
   /---------- For nextapp file ------------/
   server {
   				listen 80;
           listen [::]:80;
           
           root /var/www/html;
   
           index index.html index.htm index.nginx-debian.html;
   
           server_name myapp.com;
   
           location / {
                   proxy_pass http://localhost:3000;
                   proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                   proxy_set_header X-Forwarded-Proto $scheme;
                   proxy_set_header X-Forwarded-Port $server_port;
           }
   }
   ```

   As we can observe, notice that both files listens to port `80` but differs on `server_name` area.

   This will mean that whenever we access `myapp.com`, nginx finds the config that listens to `myapp.com` on port `80` and because we have included a `proxy_pass`, it will then redirect your request to where it has been specified, which is on our case `http://localhost:3000`. The same pattern works with `srv.myapp.com`.

   After the setup above, do not forget to reload nginx service `service nginx reload`

4. At this point, accessing `http://myapp.com` should now redirect to your Next application landing page

&nbsp;

### 5. Free SSL via Let's Encrypt!

If you want to SSL certificate for your web app for free, you can do it with Let's Encrypt.

1. Install certbot on your ubuntu by typing `apt-get install python-certbot-nginx`

2. Type in certbot command. Certbot will be the one to automatically configure your nginx files so that whenever it receives a request on port `80` it will automatically redirect you to `443` which triggers the process of automatic request/identify/signing/encryption.

   ```
   certbot --nginx -d myapp.com
   
   certbot --nginx -d srv.myapp.com
   ```

   Reload nginx, 

   ```
   service nginx reload
   ```

3. That's it!