---
title: xray搭建
date: 2024-08-17 09:32:32
tags: vpn
categories: it
---

# Xray

> nginx的分流功能  + Xray , 实现站点伪装以及公用443端口的搭建步骤
> 

<aside>
💡 xray的搭建过程需要使用证书，https默认是走443端口。
在整个搭建过程中，既想构建起xray服务，又想443端口不要被xray单独占用，
（443被占用，网站使用https协议时，就要domain:port 去访问）

</aside>

### 安装nginx

```bash
apt -y update
apt -y install curl git nginx libnginx-mod-stream
```

### 安装xray

```bash
bash <(curl -L https://raw.githubusercontent.com/XTLS/Xray-install/main/install-release.sh)
```

xray的安装就是在linux系统的几个目录添加对应的文件

```bash
(**FHS文件结构**)
installed: /etc/systemd/system/xray.service
installed: /etc/systemd/system/xray@.service
installed: /usr/local/bin/xray
installed: /usr/local/etc/xray/*.json
installed: /usr/local/share/xray/geoip.dat
installed: /usr/local/share/xray/geosite.dat
installed: /var/log/xray/access.log
installed: /var/log/xray/error.log
```

编辑nginx的主配置文件（/etc/nginx/nginx.conf）, 在http块的外部加入stream配置,
需要注意的是，因为在stream块已经使用了443端口，所以其他的nginx配置不能再使用443端口，其他nginx配置中需要443端口的配置，可以再stream添加一个upstream来进行nginx内部分流。

如下，**blue.liurunshi.work xtls**，blue.liurunshi.work这个域名分配一个xtls的upstream,流量从blue.liurunshi.work:443进来会被导向127.0.0.1:8001。
而具体的8001服务则是xray开启的一个监听端口。

```bash
stream {
        map $ssl_preread_server_name $upstream {
                blue.liurunshi.work xtls;
                web.liurunshi.work web;
        }
        
        log_format stream '$remote_addr [$time_local] [$ssl_preread_server_name] [$upstream] $status $bytes_sent $bytes_received $session_time';
        access_log /var/log/nginx/stream.log stream;
        
        upstream xtls {
                server 127.0.0.1:8001; 
        }
        upstream web {
                server 127.0.0.1:8002; 
        }
        server {
                listen 443      reuseport;
                listen [::]:443 reuseport;
                proxy_pass      $upstream;
                ssl_preread     on;
        }
}
```

搞定了nginx之后，因为vless需要有回落站点，用于伪装。所以再nginx中搭建一个网站用于回落。

```bash
cd /var/www/html
git clone https://github.com/tusenpo/FlappyFrog.git flappyfrog //可以换成一个静态页面
```

现在需要为回落站点配置nginx

```bash
/etc/nginx/conf.d/fallback.con

server {
        listen 80;
        server_name blue.liurunshi.work;
        if ($host = blue.liurunshi.work) {
                return 301 https://$host$request_uri;
        }
        return 404;
}

server {
        listen 127.0.0.1:8000;
        server_name blue.liurunshi.work;
        
        access_log /var/log/nginx/blue.liurunshi.work.log;
        
        index index.html;
        root /var/www/html/flappyfrog;
}
```

需要注意的是，回落站点是不需要配置ssl的，如果vless将请求回落到这个站点，这个站点是自动支持ssl的。而8000就是回落站点的端口，待会再xray配置的时候就需要用到这个8000端口。

证书获取，因为xray是systemd管理，systemd内的用户是nobody。需要改一下证书的权限。当然，xray官网也有使用root用户去安装xray的命令。若使用root用户安装，则不需要执行下面命令。

```bash
chown nobody:nogroup /usr/local/etc/xray/cert/cert.crt
chown nobody:nogroup /usr/local/etc/xray/cert/cert.key
```

### 配置xray规则

```bash
//先生成一个uuid，用在线生成工具生成也行
cat /proc/sys/kernel/random/uuid

/usr/local/etc/xray/config.json
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "listen": "127.0.0.1", # 仅监听在本地防止探测到下面的50001端口
            "port": 8001, # 这里的端口对应nginx内的upstream端口
            "protocol": "vless",
            "settings": {
                "clients": [
                    {
                        "id": "9fac12d0-60e0-4b0a-9bcc-c0fef59efe41", # 填写你的UUID
                        "flow": "xtls-rprx-direct",
                        "level": 0
                    }
                ],
                "decryption": "none",
                "fallbacks": [
                    {
                        "dest": "8000" # 回落站点的端口号
                    }
                 ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "xtls",
                "xtlsSettings": {
                    "alpn": [
                        "http/1.1"
                    ],
                    "certificates": [
                        {
                            "certificateFile": "/usr/local/etc/xray/cert/cert.crt", # 你的域名证书
                            "keyFile": "/usr/local/etc/xray/cert/cert.key" # 你的证书私钥
                        }
                    ]
                }
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom"
        }
    ]
}
```

配置完成之后，就能启动xray啦

```bash
systemctl enable nginx xray    //开机启动
systemctl restart nginx xray   //重启，使新配置生效
```

这里如果需要配置一个网站，并挂证书，公用443，（上面预留的8002端口。域名是 web.liurunshi.work 就排上用场了），来看看nginx的配置

```bash
server {
        listen 80;
        server_name web.liurunshi.work;
        if ($host = web.liurunshi.work) {
                return 301 https://$host$request_uri;
        }
        return 404;
}

server {
        listen 8002 ssl http2;   
        server_name web.liurunshi.work;
        
        access_log /var/log/nginx/web.liurunshi.work.log;
        
        ssl_certificate       /usr/ssl/web.liurunshi.work/cert.crt;
        ssl_certificate_key   /usr/ssl/web.liurunshi.work/cert.key;
        ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4:!DH:!DHE;
        ssl_prefer_server_ciphers on;
        
        index index.html;
        root /var/www/html/web-resume;
}
```

### v2rayN设置

![xray](../images/xray-daploy-image.png)