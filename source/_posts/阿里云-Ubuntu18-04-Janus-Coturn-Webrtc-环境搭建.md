title: 阿里云 Ubuntu18.04 Janus+Coturn Webrtc 环境搭建
author: Cyrus
date: 2020-04-06 23:14:34
tags:
---
#### 一、域名及https证书
域名跟证书都可以直接在阿里云上面弄，域名购买就默认有了。证书可以根据下面的链接获取。https://yq.aliyun.com/articles/637307


#### 二、Coturn安装
直接照网上的教程：https://blog.csdn.net/ts_dchs/article/details/97279097

简单地说几点：  
* 1、证书不用参照教程的，直接用第一步下载的证书
* 2、这里记录一下自己的配置：
~~~
listening-port=3478
listening-ip=172.18.195.84	//根据自己的情况更改
external-ip=47.107.42.156	//根据自己的情况更改
min-port=40000 
max-port=60000 
Verbose
fingerprint 
lt-cred-mech
user=cyrus:1234				//根据自己的情况更改
userdb=/usr/local/var/db/turndb
realm=cyrus.fun				//根据自己的情况更改
cert=/etc/cyrus.fun/www.cyrus.fun.pem	//根据自己的情况更改
pkey=/etc/cyrus.fun/www.cyrus.fun.key	//根据自己的情况更改
no-loopback-peers  
no-multicast-peers  
no-tcp  
no-tls  
no-cli 
~~~
* 3、启动：
~~~
turnserver -V -a -o -c /usr/local/etc/turnserver.conf
~~~

#### 三、Janus安装
照例参考网上的教程：
https://blog.csdn.net/cgs1999/article/details/89881401

简单地说几点：  
* 1、./configure 时如果 “TURN REST API client”显示为no，需要安装libcurl，不然安装成功后会一直显示Publishing。安装方法：apt-get install libcurl4-openssl-dev
* 2、编译期间如果遇到No package 'libconfig' found，可以使用apt-get install libconfig-dev 或sudo apt-get install libconfig8-dev
* 3、记录一下配置
janus.jcfg修改的地方
~~~
#支持https
certificates: {
        cert_pem = "/etc/cyrus.fun/www.cyrus.fun.pem"
        cert_key = "/etc/cyrus.fun/www.cyrus.fun.key"
        #cert_pwd = "secretpassphrase"
        #dtls_accept_selfsigned = false
        #dtls_ciphers = "your-desired-openssl-ciphers"
        #rsa_private_key = false
}

#nat: turn转发 
     turn_server = "47.107.42.156"
     turn_port = 3478
     turn_type = "udp"
     turn_user = "cyrus"
     turn_pwd = "1234"
~~~

janus.transport.http.jcfg修改的地方
~~~
general: {
        #events = true                                  # Whether to notify event handlers about transport events (default=true)
        json = "indented"                               # Whether the JSON messages should be indented (default),
                                                                        # plain (no indentation) or compact (no indentation and no spaces)
        base_path = "/janus"                    # Base path to bind to in the web server (plain HTTP only)
        http = true                                             # Whether to enable the plain HTTP interface
        port = 8088                                             # Web server HTTP port
        #interface = "eth0"                             # Whether we should bind this server to a specific interface only
        #ip = "192.168.0.1"                             # Whether we should bind this server to a specific IP address (v4 or v6) only
        https = true                                    # Whether to enable HTTPS (default=false) 支持https
        secure_port = 8089                              # Web server HTTPS port, if enabled
        #secure_interface = "eth0"              # Whether we should bind this server to a specific interface only
        #secure_ip = "192.168.0.1"              # Whether we should bind this server to a specific IP address (v4 or v6) only
        #acl = "127.,192.168.0."                # Only allow requests coming from this comma separated list of addresses
}

admin: {
        admin_base_path = "/admin"                      # Base path to bind to in the admin/monitor web server (plain HTTP only)
        admin_http = false                                      # Whether to enable the plain HTTP interface
        admin_port = 7088                                       # Admin/monitor web server HTTP port
        #admin_interface = "eth0"                       # Whether we should bind this server to a specific interface only
        #admin_ip = "192.168.0.1"                       # Whether we should bind this server to a specific IP address (v4 or v6) only
        admin_https = true                                      # Whether to enable HTTPS (default=false) 支持admin模式下的https
        admin_secure_port = 7889                        # Admin/monitor web server HTTPS port, if enabled
        #admin_secure_interface = "eth0"        # Whether we should bind this server to a specific interface only
        #admin_secure_ip = "192.168.0.1         # Whether we should bind this server to a specific IP address (v4 or v6) only
        #admin_acl = "127.,192.168.0."          # Only allow requests coming from this comma separated list of addresses
}

certificates: {
        cert_pem = "/etc/cyrus.fun/www.cyrus.fun.pem"
        cert_key = "/etc/cyrus.fun/www.cyrus.fun.key"
        #cert_pwd = "secretpassphrase"
        #ciphers = "PFS:-VERS-TLS1.0:-VERS-TLS1.1:-3DES-CBC:-ARCFOUR-128"
}
~~~
4、janus启动：
~~~
/opt/janus/bin/janus -b  //-b 表示后台运行，这里是root权限，不是的话用sudo
~~~

#### 四、安装http-server并运行janus的demo
安装
~~~
sudo apt-get install nodejs
sudo apt-get install npm
sudo npm -g install http-server

注意：
sudo: npm: command not found
解决方法：sudo apt install nodejs-legacy

安装成功可以用node -v和npm -v 验证
~~~

启动：
~~~
cd /opt/janus/share/janus/demos
http-server -S -C /etc/cyrus.fun/www.cyrus.fun.pem -K /etc/cyrus.fun/www.cyrus.fun.key
~~~

#### 五.nginx安装及运行janus的demo
如果不想用http-server，也可以用nginx运行。参照网上教程：https://blog.csdn.net/cgs1999/article/details/89881733
安装：
~~~
sudo apt-get install nginx -y
~~~

配置： vi /etc/nginx/conf.d/default.conf
~~~
server {
        listen       8080;
        listen  *:443  ssl;
        server_name  localhost;

        location / {
                root /opt/janus/share/janus/demos;
                index index.html index.htm index.php;
        }

        location /janus {
                proxy_set_header Host $host:$server_port;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_pass https://172.18.195.84:8089/janus;
        }

        location /admin {
                proxy_set_header Host $host:$server_port;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_pass https://172.18.195.84:7089/admin;
        }

        location ~ /.*\.(bmp|gif|jpg|png|css|js|cur|flv|ico|swf|doc|pdf|html)$ {
                root /opt/janus/share/janus/demos;
                expires 1d;
        }
        

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
                root   /usr/share/nginx/html;
        }

        #ssl_certificate /etc/nginx/ssl/nginx.crt;
        #ssl_certificate_key /etc/nginx/ssl/nginx.key;
        ssl_certificate /etc/cyrus.fun/www.cyrus.fun.pem;
        ssl_certificate_key /etc/cyrus.fun/www.cyrus.fun.key;
}
~~~

运行：service nginx start



