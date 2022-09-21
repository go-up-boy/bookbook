# 部署 Nginx、PHP 环境

## 方式一
1. php 和 nginx 安装
```shell
docker pull php:7.4-fpm
docker pull nginx
```
2. 运行php
    * 注意容器内路径
```shell
docker run -d --privileged=true -v C:\Users\z\nginx\www:/usr/share/nginx/html --name php -p 9000:9000 php:7.4-fpm
```
3. 运行nginx
    * 注意这里 映射容器内的 /usr/share/nginx/html 目录要与PHP容器内目录一致
    * --link 连接的 容器名:别名，也可以通过 docker inspect 查看容器ip
```shell
docker run -d -p 80:80 --privileged=true -v C:\Users\z\nginx\www:/usr/share/nginx/html -v C:\Users\z\nginx\conf:/etc/nginx/conf.d --name nginx --link php:php nginx:latest
```
4. nginx 网站配置
```shell
server {
    listen       80;
    root   /usr/share/nginx/html;
    server_name  localhost; #这里修改成自己的域名，我这里是本地运行所以填的localhost
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm index.php;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    location ~ \.php$ {
        include        fastcgi_params;
        fastcgi_pass   php:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

## 方式二 自行构建 php-nginx镜像