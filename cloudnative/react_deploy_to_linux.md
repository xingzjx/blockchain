# react 部署到 linux 服务器


# 打包上传到服务器

打包

```
npm build

```

然后，上传　build　目录到服务器


# 安装 nginx

```sh

sudo apt-get install nginx

# 开机自启动
systemctl start nginx.service  
systemctl enable nginx.service

```

配置 nginx.conf 文件

```conf

worker_processes 1;
events {
    worker_connections 1024;
}

http {
    include mime.types;
    default_type application/octet-stream;
    sendfile on;
    keepalive_timeout 65;

    include /etc/nginx/vhost/*.conf;

    server {
         listen 80;
         server_name _;
         root /home/www/web;
    }
}


```

检查配置

```sh

sudo nginx -t -c /etc/nginx/nginx.conf

```

重新加载配置

```
sudo nginx -s reload

```




