## Guacamole Ubuntu Docker

<pre>sudo mkdir -p /app/{images/guacamole,guacamole/{home/extensions,db/{data,conf.d}}}

</pre>

- #### Maria DB

<pre>sudo bash -c 'cat <<EOT > /app/compose/guacamole.yml
version: "3"
services:
 guacamole-db:
  image: mariadb:latest
  container_name: "guacamole-db"
  volumes:
   - "/app/guacamole/db/data:/var/lib/mysql"
   - "/app/guacamole/db/conf.d:/etc/mysql/conf.d"
  environment:
   MYSQL_ROOT_PASSWORD: "PASSWORD"
  network_mode: bridge
  logging:
   driver: "json-file"
   options:
    max-size: "200k"
    max-file: "5"
  restart: always
EOT' && \
docker-compose -f /app/compose/guacamole.yml config

docker-compose -p guacamole -f /app/compose/guacamole.yml up -d
</pre>

<pre><i>// pass and sock no, all yes</i>

docker exec -it guacamole-db mysql_secure_installation
</pre>

<pre>bash -c "cat <<EOT > ~/guacamole.sql
CREATE DATABASE guacamole;
CREATE USER 'guacadmin'@'%' IDENTIFIED BY 'PASSWORD';
GRANT SELECT,INSERT,UPDATE,DELETE ON \\\`guacamole\\\`.* TO 'guacadmin'@'%';
FLUSH PRIVILEGES;
EOT"

docker cp ~/guacamole.sql guacamole-db:/tmp/ && \
docker exec -it guacamole-db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" < /tmp/guacamole.sql'
</pre>

<pre>docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > ~/initdb.sql

docker cp ~/initdb.sql guacamole-db:/tmp/ && \
docker exec -it guacamole-db sh -c 'cat /tmp/initdb.sql | mysql -u root -p"$MYSQL_ROOT_PASSWORD" guacamole'
</pre>
  
<pre>rm -rf ~/{initdb,guacamole}.sql && docker exec -it guacamole-db rm -rf /tmp/{initdb,guacamole}.sql

docker exec -it guacamole-db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW DATABASES;"' && \
docker exec -it guacamole-db sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW GRANTS FOR guacadmin;"'  
</pre>

<pre>sudo nano /app/guacamole/db/conf.d/my.cnf
<i>
[mysqld]
skip-name-resolve
skip-external-locking
default-time-zone=+05:00
</i></pre>

<pre>docker exec -it guacamole-db mysql -u root -p
<i>
use mysql;
select Host, User from user;
delete from user where User="root" and Host="************";
select Host, User from proxies_priv;
delete from proxies_priv where User="root" and Host="************";
flush privileges;
quit
</i>
docker-compose -p guacamole -f /app/compose/guacamole.yml down
</pre>

- #### Branding Guacamole

<pre>wget https://github.com/DeAlexPesh/ubuntu-guacamole/raw/main/branding.jar

sudo mv branding.jar /app/guacamole/home/extensions/branding.jar
sudo chown -R root:root /app/guacamole/home/* && \
sudo chmod -R 755 /app/guacamole/home/*
</pre>
  
- #### LDAP Guacamole

<pre>wget https://downloads.apache.org/guacamole/1.3.0/binary/guacamole-auth-ldap-1.3.0.tar.gz -P ~ && \
tar -xzf guacamole-auth-ldap-1.3.0.tar.gz && rm guacamole-auth-ldap-1.3.0.tar.gz && \
sudo mv ~/guacamole-auth-ldap-1.3.0/guacamole-auth-ldap-1.3.0.jar /app/guacamole/home/extensions/guacamole-auth-ldap-1.3.0.jar && \
sudo mv ~/guacamole-auth-ldap-1.3.0/schema /app/guacamole/home/extensions/schema && \
rm -rf ~/guacamole-auth-ldap-1.3.0.tar.gz && \
rm -rf ~/guacamole-auth-ldap-1.3.0
</pre>

<pre>sudo nano /app/guacamole/home/guacamole.properties
<i>
enable-environment-properties: true
# LDAP properties
ldap-hostname: dc01.domain.ru
ldap-port: 389
ldap-username-attribute: sAMAccountName
ldap-encryption-method: none
ldap-search-bind-dn: cn=guacadmin,ou=IT,dc=domain,dc=ru
ldap-search-bind-password: PASSWORD
ldap-user-base-dn: dc=domain,dc=ru
ldap-user-search-filter: (&(memberOf=cn=desk,ou=IT,dc=domain,dc=ru))
ldap-max-search-results: 400
</i></pre>

<pre>sudo chown -R root:root /app/guacamole/home/* && \
sudo chmod -R 755 /app/guacamole/home/*
</pre>

<i>`guacadmin - user with access to AD`</i></br>
<i>`desk - group of users who can use guacamole`</i>
  
- #### Guacamole
  
<pre>sudo nano /app/images/guacamole/Dockerfile
<i>
FROM guacamole/guacamole:latest
RUN sed -i 's/redirectPort="8443"/redirectPort="8443" server="" secure="true"/g' /usr/local/tomcat/conf/server.xml \
 && sed -i 's/<Server port="8005" shutdown="SHUTDOWN">/<Server port="-1" shutdown="SHUTDOWN">/g' /usr/local/tomcat/conf/server.xml \
 && rm -rf /usr/local/tomcat/webapps/{docs,examples,manager,host-manager} \
 && chmod -R 400 /usr/local/tomcat/conf
</i></pre>

<pre>sudo bash -c 'cat <<EOT >> /app/compose/guacamole.yml

 guacd:
  image: guacamole/guacd:latest
  container_name: "guacd"
  network_mode: bridge
  dns:
   - 192.168.1.*
  dns_search: domain.ru
  logging:
   driver: "json-file"
   options:
    max-size: "200k"
    max-file: "5"
  restart: always

 guacamole:
  build: /app/images/guacamole
  image: images/guacamole:mode
  container_name: "guacamole"
  ports:
   - "9011:8080"
  links:
   - guacd:guacd
   - guacamole-db:mysql
  volumes:
   - "/app/guacamole/home:/.guacamole"
  environment:
   MYSQL_DATABASE: "guacamole"
   MYSQL_USER: "guacadmin"
   MYSQL_PASSWORD: "PASSWORD"
   GUACAMOLE_HOME: "/.guacamole"
   EXTENSIONS: "auth-ldap"
  depends_on:
   - guacd
   - guacamole-db
  network_mode: bridge
  dns:
   - 192.168.1.2
  dns_search: domain.ru
  logging:
   driver: "json-file"
   options:
    max-size: "200k"
    max-file: "5"
  restart: always
EOT' && \
docker-compose -f /app/compose/guacamole.yml config

docker-compose -p guacamole -f /app/compose/guacamole.yml up -d
</pre>

<pre>sudo sed -i "34 s|^|#|" /app/compose/guacamole.yml && \
docker rmi guacamole/guacamole:latest
</pre>

<i>`http://192.168.1.*:9011/guacamole/#/settings/sessions`</i></br>
<i>`guacadmin | guacadmin`</i>

- #### Nginx proxy

<pre>sudo nano /app/images/guacamole/Dockerfile
<i>
FROM guacamole/guacamole:latest
RUN sed -i -e ':a;N;/>/!ba' -e 's|^\s*<Connector port="8080" protocol="HTTP/1.1".*/>| \
   <Connector port="8080" protocol="HTTP/1.1"\n \
              connectionTimeout="20000"\n \
              URIEncoding="UTF-8"\n \
              redirectPort="8443" />\n\n \
|g' /usr/local/tomcat/conf/server.xml \
 && sed -i 'N;s|^\s*</Host>|\n \
       <Valve className="org.apache.catalina.valves.RemoteIpValve"\n \
              internalProxies="192.168.1.0"\n \
              remoteIpHeader="x-forwarded-for"\n \
              remoteIpProxiesHeader="x-forwarded-by"\n \
              protocolHeader="x-forwarded-proto" />\n\n \
|g' /usr/local/tomcat/conf/server.xml \
 && sed -i 's/<Server port="8005" shutdown="SHUTDOWN">/<Server port="-1" shutdown="SHUTDOWN">/g' /usr/local/tomcat/conf/server.xml \
 && rm -rf /usr/local/tomcat/webapps/{docs,examples,manager,host-manager} \
 && chmod -R 400 /usr/local/tomcat/conf
</i></pre>

`docker exec -it guacamole sh -c 'cat /usr/local/tomcat/conf/server.xml'`</br>

<details>
<summary>/usr/local/tomcat/conf/server.xml</summary>
<pre><Connector port="8080" protocol="HTTP/1.1" 
           connectionTimeout="20000"
           URIEncoding="UTF-8"
           redirectPort="8443" />
          
<Host ... >
<Valve className="org.apache.catalina.valves.RemoteIpValve"
       internalProxies="192.168.1.0"
       remoteIpHeader="x-forwarded-for"
       remoteIpProxiesHeader="x-forwarded-by"
       protocolHeader="x-forwarded-proto" />
</Host>
</pre></details>

<pre>guacamole.conf
<i>
upstream http_guac {
  ip_hash;
  server 192.168.1.*:9011;
}

server {
  listen 443 ssl http2;
  charset utf-8;

  server_name desk.domain.ru;

  access_log /var/log/nginx/desk.access.log main;
  error_log /var/log/nginx/desk.error.log warn;

  include /etc/nginx/certs/ssl.conf;

  ssl_certificate /etc/nginx/certs/ssl/live/domain.ru/fullchain.pem;
  ssl_trusted_certificate /etc/nginx/certs/ssl/live/domain.ru/chain.pem;
  ssl_certificate_key /etc/nginx/certs/ssl/live/domain.ru/privkey.pem;

  add_header X-Content-Type-Options nosniff;
  add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";

  proxy_hide_header X-Frame-Options;
  proxy_hide_header Expires;

  location / {
    proxy_http_version 1.1;
    proxy_buffering off;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "";
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Authorization "";

    proxy_cookie_path /guacamole/ /;
    proxy_pass http://http_guac/guacamole/;
  }
  error_page 401 404 500 502 503 504 /fail.html;
  location = /fail.html {
    root /usr/share/nginx/html/files_app/fails;
    internal;
  }
}
</i></pre>
