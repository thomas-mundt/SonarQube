# Install SonarQube

## Description

It's every developer's dream to have clean and issue-free code which can readily be deployed into staging and production environments. One tool that can help you achieve this is in your CI/CD pipeline is SonarQube. SonarQube is a cross-platform and web-based tool used for continuous inspection of source code. It is written in Java. SonarQube enables you to write cleaner and safer code by inspecting code and detecting bugs and other inconsistencies.

SonarQube can be integrated into platforms such as GitHub, Gitlab, BitBucket, and Azure DevOps to mention a few platforms. It comes in various editions including Community, Developer, Enterprise, and Datacenter editions.

## Setup

### Ubuntu 20.04

Prerequisites
```
1. Ubuntu 20.04 LTS with a sudo user configured.
2. Ensure your system has a minimum of 4GB RAM and 2vCPU cores
```


Install
```
sudo apt update
sudo apt install net-tools unzip vim curl

sudo vim /etc/sysctl.conf
#Add the following lines
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096

sudo vim /etc/security/limits.conf
#At the very bottom, add the following lines
sonarqube - nofile 65536
sonarqube - nproc 4096

```


Install OpenJDK
```
sudo apt install openjdk-11-jdk
```


Install PostgreSQL database 
SonarQube dropped support for MySQL and now only supports PostgreSQL
```
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo apt update

sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql
sudo systemctl status postgresql

sudo netstat -pnltu | grep 5432

```


Configure PostgreSQL
```
sudo passwd postgres
su - postgres
createuser sonar
psql
ALTER USER sonar WITH ENCRYPTED PASSWORD 'strong_password';
CREATE DATABASE sonarqube OWNER sonar;
GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;
\q
```


Download and configure SonarQube
```
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.0.1.46107.zip 
unzip sonarqube-9.0.1.46107.zip
sudo mv sonarqube-9.0.1.46107 /opt/sonarqube

```

Create new user and group
```
sudo groupadd sonar
sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
sudo chown -R sonar:sonar /opt/sonarqube/

```


Configure SonarQube
```
sudo vim  /opt/sonarqube/conf/sonar.properties

#Locate and uncomment the following lines
sonar.jdbc.username=
sonar.jdbc.password=


#These represent the SonarQube database user and password that we created in the PostgreSQL database server. Therefore, fill in the values accordingly.
sonar.jdbc.username=sonar_user
sonar.jdbc.password=strong_password

#Next, modify these lines so that they look as what is provided
sonar.jdbc.url=jdbc:postgresql://localhost:5432/sonarqube
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:MaxDirectMemorySize=256m -XX:+HeapDumpOnOutOfMemoryError

#Thereafter, modify the following lines to appear as they look.
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.web.javaAdditionalOpts=-server
sonar.log.level=INFO
sonar.path.logs=logs

#Next, modify the user that will run the SonarQube service by editing the file shown
sudo vim /opt/sonarqube/bin/linux-x86-64/sonar.sh
RUN_AS_USER=sonar
```

Create a Systemd service file for SonarQube
```
sudo vim  /etc/systemd/system/sonarqube.service

[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
sudo systemctl status sonarqube
```

```
sudo ufw allow '9000'
sudo netstat -pnltu | grep 9000
```


```
http://<server-ip>:9000/
admin
admin
```


## Install and Configure Nginx with SSL (optional)

```
sudo apt install nginx

sudo systemctl enable nginx
sudo systemctl start nginx
```

sudo vim  /etc/nginx/sites-available/sonarqube.conf
```
server {

listen 80;
server_name example.com or SERVER-IP;
access_log /var/log/nginx/sonar.access.log;
error_log /var/log/nginx/sonar.error.log;
proxy_buffers 16 64k;
proxy_buffer_size 128k;

location / {
proxy_pass http://127.0.0.1:9000;
proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
proxy_redirect off;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto http;
}
}
```


```
sudo ln -s /etc/nginx/sites-available/sonarqube.conf  /etc/nginx/sites-enabled/sonarqube.conf
sudo nginx -t

sudo systemctl restart nginx
sudo ufw allow 'Nginx Full'
sudo ufw --reload

#You can now access your SonarQube with through its domain name

```

Here, we are going to use the free let's encrypt certificate. To configure that we need to run cerbot for Nginx 

sudo certbot --nginx
```
Saving debug log to /var/log/letsencrypt/letsencrypt.log
 Plugins selected: Authenticator nginx, Installer nginx
 Enter email address (used for urgent renewal and security notices) (Enter 'c' to
 cancel): alain@websitefortesting.com                                                    
 
 Please read the Terms of Service at
 https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
 agree in order to register with the ACME server at
 https://acme-v02.api.letsencrypt.org/directory
 
 (A)gree/(C)ancel: A
 
 Would you be willing to share your email address with the Electronic Frontier
 Foundation, a founding partner of the Let's Encrypt project and the non-profit
 organization that develops Certbot? We'd like to send you email about our work
 encrypting the web, EFF news, campaigns, and ways to support digital freedom.
 
 (Y)es/(N)o: N
Saving debug log to /var/log/letsencrypt/letsencrypt.log
 Plugins selected: Authenticator nginx, Installer nginx
 Which names would you like to activate HTTPS for?
 
 1: websitefortesting.com
 
 Select the appropriate numbers separated by commas and/or spaces, or leave input
 blank to select all options shown (Enter 'c' to cancel): 1
 Obtaining a new certificate
 Performing the following challenges:
 http-01 challenge for websitefortesting.com
 Waiting for verification…
 Cleaning up challenges
 Deploying Certificate to VirtualHost /etc/nginx/sites-enabled/sonarqube.conf
 Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
 
 1: No redirect - Make no further changes to the webserver configuration.
 2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
 new sites, or if you're confident your site works on HTTPS. You can undo this
 change by editing your web server's configuration.
 
 Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2
 Redirecting all traffic on port 80 to ssl in /etc/nginx/sites-enabled/sonarqube.conf
 
 Congratulations! You have successfully enabled https://websitefortesting.com
 You should test your configuration at:
 https://www.ssllabs.com/ssltest/analyze.html?d=websitefortesting.com
 
 IMPORTANT NOTES:
 Congratulations! Your certificate and chain have been saved at:
 /etc/letsencrypt/live/websitefortesting.com/fullchain.pem
 Your key file has been saved at:
 /etc/letsencrypt/live/websitefortesting.com/privkey.pem
 Your cert will expire on 2021-11-27. To obtain a new or tweaked
 version of this certificate in the future, simply run certbot again
 with the "certonly" option. To non-interactively renew all of
 your certificates, run "certbot renew"
 If you like Certbot, please consider supporting our work by:
 Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 Donating to EFF:                    https://eff.org/donate-le 
```


By default, let's encrypt will add some lines in the Nginx server block file.
```
server {
         server_name websitefortesting.com;
         add_header Strict-Transport-Security max-age=2592000;
         #rewrite ^ https://$server_name$request_uri? permanent;
         access_log  /var/log/nginx/sonarqube.access.log;
         error_log   /var/log/nginx/sonarqube.error.log;
     proxy_buffers 16 64k;     
           proxy_buffer_size 128k;     

           location / {
             proxy_pass http://127.0.0.1:9000;            
             proxy_set_header Host $host;             
             proxy_set_header X-Real-IP $remote_addr;             
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;             
             proxy_set_header X-Forwarded-Proto http;     
           } 
          listen 443 ssl; # managed by Certbot 
          ssl_certificate /etc/letsencrypt/live/websitefortesting.com/fullchain.pem; # managed by Certbot 
           ssl_certificate_key /etc/letsencrypt/live/websitefortesting.com/privkey.pem; # managed by Certbot 
          include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot 
          ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
 }
 server {
     if ($host = websitefortesting.com) {
         return 301 https://$host$request_uri;
     } # managed by Certbot
     
            listen 80;     
            server_name websitefortesting.com; return 404; # managed by Certbot

 }
```


Now you can access SonarQube securely with HTTPS URL configured with let's encrypt
```
https://domain-name
```



