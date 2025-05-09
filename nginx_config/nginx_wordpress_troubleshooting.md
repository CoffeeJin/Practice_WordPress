# 🧾 故障记录：WordPress Behind Nginx Reverse Proxy – 无法访问排查文档

## 📌 问题概述

在将 WordPress 部署在内网主机（192.168.100.32:8080），并通过公网 Nginx 网关实现 HTTPS 反向代理（chatbot.speedy.com）时，外部访问失败，浏览器提示：

> “This site can’t be reached”  
> 或 curl 报错：**SSL certificate problem: unable to get local issuer certificate**

## 🧠 根本原因总结

| 问题编号 | 问题描述 |
|----------|----------|
| ❌ 1 | Nginx 配置的证书 `speedy.com.crt` 缺少中间证书，导致客户端验证失败 |
| ❌ 2 | WordPress 没有识别请求为 HTTPS，触发错误跳转回内网 IP（HTTP 301）|
| ✅ 3 | 反向代理配置未显式声明 `X-Forwarded-Proto`，影响 WordPress 协议判断 |

## 🛠️ 排查步骤与操作记录

### ✅ 第一步：确认域名解析与端口监听

```bash
ping chatbot.speedy.com
ss -tlnp | grep 443
```

### ✅ 第二步：使用 curl 检测 HTTPS 错误原因

```bash
curl -v https://chatbot.speedy.com
```

返回错误：
```
SSL certificate problem: unable to get local issuer certificate
```

### ✅ 第三步：查看证书签发机构

```bash
openssl s_client -connect chatbot.speedy.com:443 -servername chatbot.speedy.com </dev/null 2>/dev/null | openssl x509 -noout -issuer
```

结果：
```
issuer = C = US, O = Let's Encrypt, CN = R11
```

### ✅ 第四步：下载中间证书并拼接 fullchain

```bash
wget http://r11.i.lencr.org/ -O lets-encrypt-r11.pem
cat speedy.com.crt lets-encrypt-r11.pem | sudo tee speedy.com.fullchain.crt > /dev/null
```

### ✅ 第五步：修改 Nginx 配置使用 fullchain

```nginx
ssl_certificate     /etc/nginx/ssl/speedy.com.fullchain.crt;
ssl_certificate_key /etc/nginx/ssl/speedy.com.key;
```

重载：

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### ✅ 第六步：防止 WordPress 重定向回 HTTP 或内网地址

`wp-config.php` 中添加：

```php
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}
```

### ✅ 第七步：标准反向代理配置样例

```nginx
location / {
    proxy_pass http://192.168.100.32:8080;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
}
```

## 🧩 附加建议（未来安全加固）

- 禁止访问 `.git`, `.env` 等敏感路径；
- 部署 fail2ban 防扫描；
- 考虑将子域加入 Cloudflare 或其他 WAF；
- 配置证书自动续期（Let's Encrypt + cron 或 certbot）。
