# 使用 Caddy + Gost 的代理方案

一直在使用已故大佬左耳朵的 [科学上网教程](https://haoel.github.io) ，近期做了一点小改动，记录一下。主要就是利用caddy的证书自动化刷新能力，简化证书的配置。

## 一、在宿主机安装 Caddy

### Ubuntu/Debian:
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```

## 二、配置 Caddy

创建 Caddy 配置文件 `/etc/caddy/Caddyfile`:

```caddyfile
{
    # 全局配置
    email your-email@example.com

    # 证书存储位置（默认在 /var/lib/caddy/.local/share/caddy）
    # Caddy 会自动管理这个目录
}

# Web 服务 - 运行在 8443 端口
https://your-domain.com:8443 {
    # 提供静态网站
    root * /var/www/html
    file_server
}

# HTTP 自动重定向到 HTTPS (Web服务)
http://your-domain.com {
    redir https://your-domain.com:8443{uri}
}
```

## 三、启动 Caddy 并验证证书

```bash
# 启动 Caddy
sudo systemctl start caddy
sudo systemctl enable caddy

# 查看状态
sudo systemctl status caddy

# 等待几秒，让 Caddy 申请证书
sleep 10

# 查看证书位置
sudo caddy list-certificates
```

Caddy 会将证书保存在: `/var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/your-domain.com/`

## 四、配置 Gost Docker 脚本

创建 `start-gost.sh`:

```bash
#!/bin/bash

DOMAIN="your-domain.com" # <- 自定义
USER="username" # <- 自定义
PASS="password" # <- 自定义
PORT=443

# Caddy 默认证书存储位置
CADDY_CERT_BASE="/var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory"
CERT_DIR="${CADDY_CERT_BASE}/${DOMAIN}"
CERT="${CERT_DIR}/${DOMAIN}.crt"
KEY="${CERT_DIR}/${DOMAIN}.key"

sudo docker stop gost 2>/dev/null || true
sudo docker rm gost 2>/dev/null || true

echo "启动 Gost 容器..."
sudo docker run -d --name gost \
    --restart unless-stopped \
    -v ${CADDY_CERT_BASE}:${CADDY_CERT_BASE}:ro \
    --net=host \
    gogost/gost \
    -L "http2://${USER}:${PASS}@0.0.0.0:${PORT}?cert=${CERT}&key=${KEY}&probe_resist=code:404&knock=www.google.com"
```

## 启动服务

``` 
chmod +x start-gost.sh
sudo ./start-gost.sh

```

## 改进

1.  **证书自动管理**: Caddy 自动申请和续期，无需 certbot 和 cron
2.  **同时提供 Web 服务**: 看起来像正常网站
3.  **8443提供web服务**: 没什么大用，可以检查一下服务状态

