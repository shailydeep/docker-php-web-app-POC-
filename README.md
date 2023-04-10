# docker-php-web-app-POC-
Step 1 – Create an Nginx Container
Before starting, you will need to create and launch an Nginx container to host the PHP application.

First, create a directory for your project with the following command:

mkdir ~/docker-project
Next, change the directory to your project and create a docker-compose.yml file to launch the Nginx container.

       cd ~/docker-project
       nano docker-compose.yml
Add the following lines:

nginx:   
      image: nginx:latest  
      container_name: nginx-container  
      ports:   
       - 80:80 
it will make sure nginx container is running on port 80. Save and close the file when you are finished.

Next, launch the Nginx container with the following command:

docker-compose up -d
You can check the running container with the following command:

 docker ps
You should see the following output:

CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS                               NAMES
c6641e4d5bbf   nginx:latest   "/docker-entrypoint.…"   5 seconds ago   Up 3 seconds   0.0.0.0:80->80/tcp, :::80->80/tcp   nginx-container
Now, open your web browser and access your Nginx container using the URL http://your-server-ip. You should see the Nginx test page on the following screen:
image

Step 2 – Create a PHP Container
First, create a new directory inside your project with the following command:

mkdir  ~/docker-project/php_code
Download PHP code (its a e-commerce simple code )

 git clone https://github.com/kodekloudhub/learning-app-ecommerce ~/docker-project/php_code/
here paste all php data that along with index.php

First, you will need to modify the PHP image and install the PHP extension for mariadb in order to connect to the mariadb database.

First, create a Dockerfile for PHP with the following command:

     nano  ~/docker-project/php_code/Dockerfile
Add the following lines:

      FROM php:7.0-fpm  
      RUN docker-php-ext-install mysqli pdo pdo_mysql
      RUN docker-php-ext-enable mysqli
Save and close the file.

create a directory for Nginx inside your project directory:

 mkdir ~/docker-project/nginx
Next, create an Nginx default configuration file to run your PHP application:

 nano ~/docker-project/nginx/default.conf
Add the following lines:

server {  

     listen 80 default_server;  
     root /var/www/html;  
     index index.html index.php;  

     charset utf-8;  

     location / {  
      try_files $uri $uri/ /index.php?$query_string;  
     }  

     location = /favicon.ico { access_log off; log_not_found off; }  
     location = /robots.txt { access_log off; log_not_found off; }  

     access_log off;  
     error_log /var/log/nginx/error.log error;  

     sendfile off;  

     client_max_body_size 100m;  

     location ~ .php$ {  
      fastcgi_split_path_info ^(.+.php)(/.+)$;  
      fastcgi_pass php:9000;  
      fastcgi_index index.php;  
      include fastcgi_params;
      fastcgi_read_timeout 300;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;  
      fastcgi_intercept_errors off;  
      fastcgi_buffer_size 16k;  
      fastcgi_buffers 4 16k;  
    }  

     location ~ /.ht {  
      deny all;  
     }  
    } 

Save and close the file.

Next, create a Dockerfile inside the nginx directory. This will copy the Nginx default config file to the Nginx container.

   nano ~/docker-project/nginx/Dockerfile
Add the following lines:

   FROM nginx

   COPY ./default.conf /etc/nginx/conf.d/default.conf
Next, edit the docker-compose.yml file:

 nano ~/docker-project/docker-compose.yml
Remove the old contents and add the following contents:


version: "3.9"
services:
   nginx:
     build: ./nginx/
     ports:
       - 80:80
  
     volumes:
         - ./php_code/:/var/www/html/

   php:
     build: ./php_code/
     expose:
       - 9000
     volumes:
        - ./php_code/:/var/www/html/
Now, launch the container with the following command:

      cd ~/docker-project
      docker-compose up -d
You can verify the running containers with the following command:

  docker ps
Now, open your web browser and access the URL http://your-server-ip or localhot . You should see your php web content

but here  is something wrong when you invoke url your php is running properly but output is not that we want  we want fetch data from database and display it so lets create database.

Step 3 – Create a Mariadb Container
In this section, we will create a mariadb database container and linked it to all other containers.

Then, edit the docker-compose.yml file to create entry for a mariadb container

         nano ~/docker-project/docker-compose.yml
Make the following changes:


version: "3.9"
services:
   nginx:
     build: ./nginx/
     ports:
       - 80:80
  
     volumes:
         - ./php_code/:/var/www/html/

   php:
     build: ./php_code/
     expose:
       - 9000
     volumes:
        - ./php_code/:/var/www/html/


   db:    
      image: mariadb  
      volumes: 
        -    mysql-data:/var/lib/mysql
      environment:  
       MYSQL_ROOT_PASSWORD: mariadb
       MYSQL_DATABASE: ecomdb 


volumes:
    mysql-data:
Configure Database
Take a CLI of db by following command:

           docker exec -it [db container id or name ] /bin/sh
Enter in mariadb:

   mysql -u root -pmariadb  
Create user in db:

     CREATE USER 'shaily'@'%'  IDENTIFIED BY "shaily@10";
Provide ALL PRIVILEGES to user :

    GRANT ALL PRIVILEGES on *.* to 'shaily'@'%';
    FLUSH PRIVILEGES;
back to db bash :

    exit 
Load Product Inventory Information to database


cat > db-load-script.sql <<-EOF
USE ecomdb;
CREATE TABLE products (id mediumint(8) unsigned NOT NULL auto_increment,Name varchar(255) default NULL,Price varchar(255) default NULL, ImageUrl varchar(255) default NULL,PRIMARY KEY (id)) AUTO_INCREMENT=1;

INSERT INTO products (Name,Price,ImageUrl) VALUES ("Laptop","100","c-1.png"),("Drone","200","c-2.png"),("VR","300","c-3.png"),("Tablet","50","c-5.png"),("Watch","90","c-6.png"),("Phone Covers","20","c-7.png"),("Phone","80","c-8.png"),("Laptop","150","c-4.png");

EOF

Run sql script

       mysql -u root -pmariadb< db-load-script.sql
we mounted a volume in db /var/lib/mysql so our data is loaded and persistent.

NOTE:->


'deepak'@'%' :-> 'user'@'any host'

*.*.         :-> all databases . all tables 



Exit bash of db:

       exit
Update index.php
Update index.php file to connect to the right database server.

  sudo sed -i 's/172.20.1.101/db/g' /var/www/html/index.php

 <snap>
<?php
                  $link = mysqli_connect('db', 'deepak', 'deepak123', 'ecomdb');
                  if ($link) {
                  $res = mysqli_query($link, "select * from products;");
                  while ($row = mysqli_fetch_assoc($res)) { ?>

 <snap>
so, it can find database for by db name . because all container are communicate by name of container in same natwork.

chek URL and refresh it........
