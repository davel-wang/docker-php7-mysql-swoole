# docker-php7-mysql-swoole安装说明

## 安装及启动docker
```
    yum -y install docker

    #镜像加速
    echo "DOCKER_OPTS=\"\$DOCKER_OPTS --registry-mirror=http://hub-mirror.c.163.com\"" >> /etc/default/docker

    #报错修复
    vi /etc/sysconfig/docker

    -selinux-enabled 改为 --selinux-enabled=false

    #启动docker
    systemctl start docker

```


## 正式步骤

### docker nginx
```
docker pull nginx
docker run --name nginx-test -p 8081:80 -d nginx

mkdir -p ~/nginx/www ~/nginx/logs ~/nginx/conf
mkdir ~/nginx/conf/conf.d 
cd ~/nginx/conf/conf.d 
touch test-php.conf
vi test-php.conf

server {
    listen       80;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm index.php;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location ~ \.php$ {
        fastcgi_pass   php:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  /www/$fastcgi_script_name;
        include        fastcgi_params;
    }
}

#查找生成的nginx容器ID
docker ps -a
docker cp 72763ce53f25:/etc/nginx/nginx.conf ~/nginx/conf #72763ce53f25容器ID

```


### docker mysql
```
docker pull mysql:5.7
mkdir -p ~/mysql/data ~/mysql/logs ~/mysql/conf

#启动mysql
docker run -p 3306:3306 --name dockermysql57 -v ~/mysql/conf:/etc/mysql/conf.d -v ~/mysql/logs:/logs -v ~/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7

```


### docker php7.2
```

docker pull php:7.2-fpm
docker run -p 9001:9001 --name php72fpm -v ~/nginx/www:/www --link dockermysql57 -d php:7.2-fpm

#查找生成的php容器ID
docker ps -a

#进入容器添加PHP扩展
docker exec -i -t 47b5d91b1a17 bash #47b5d91b1a17容器ID

#安装swoole扩展(选择等2.1.1版本,更高版本会报错)
pecl install http://pecl.php.net/get/swoole-2.1.1.tgz
docker-php-ext-enable swoole

#安装reids扩展(选择等3.1.1版本,更高版本会报错)
pecl install http://pecl.php.net/get/redis-3.1.1.tgz
docker-php-ext-enable redis

#安装pdo_mysql扩展
docker-php-ext-install pdo_mysql
docker-php-ext-enable pdo_mysql

```


生成新的镜像
```
docker commit 47b5d91b1a17 davelswang/php7-mysql-swoole

```


### 根据新的镜像重新生成容器
```

docker run -p 9000:9000 --name dockerphp7 -v ~/nginx/www:/www --link dockermysql57 -d davelswang/php7-mysql-swoole

```


### 启动nginx
```
cd ~/nginx/www/
touch index.php
vi index.php

<?php
phpinfo();
?>

#启动nginx
docker run --name dockernginx -p 80:80 -d \
    -v ~/nginx/www:/usr/share/nginx/html:ro \
    -v ~/nginx/conf/conf.d:/etc/nginx/conf.d:ro \
    --link dockerphp7:php \
    --link dockermysql57:mysql \
    nginx

```


## 重启后的步骤
```
#启动mysql
docker run -p 3306:3306 --name dockermysql57 -v ~/mysql/conf:/etc/mysql/conf.d -v ~/mysql/logs:/logs -v ~/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7

#启动php
docker run -p 9000:9000 --name dockerphp7 -v ~/nginx/www:/www --link dockermysql57 -d davelswang/php7-mysql-swoole

#启动nginx
docker run --name dockernginx -p 80:80 -d \
    -v ~/nginx/www:/usr/share/nginx/html:ro \
    -v ~/nginx/conf/conf.d:/etc/nginx/conf.d:ro \
    --link dockerphp7:php \
    --link dockermysql57:mysql \
    nginx


```