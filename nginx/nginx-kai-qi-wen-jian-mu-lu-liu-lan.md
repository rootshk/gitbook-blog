# Nginx 开启文件目录浏览

&#x20;**Nginx配置**

```
server {
    listen 80;
    server_name file.roothk.top;
    access_log /app/nginx/logs/file-nginx-access.log main;
    error_log /app/nginx/logs/file-nginx-error.log;

    location / {
        auth_basic "Please enter your username and password";
        auth_basic_user_file /app/nginx/conf/file_passwd;
        #需要下载的文件存放的目录
        alias "/sftp/";
        sendfile on;
        autoindex on;  # 开启目录文件列表
        autoindex_exact_size on;  # 显示出文件的确切大小，单位是bytes
        autoindex_localtime on;  # 显示的文件时间为文件的服务器时间
        charset utf-8,gbk;  # 避免中文乱码
    }
}
```

**密码生成**

{% embed url="https://www.sojson.com/htpasswd.html" %}

> 选择 Crypt (all Unix servers)
