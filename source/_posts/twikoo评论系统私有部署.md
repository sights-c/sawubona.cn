---
title: Twikoo评论系统私有部署
date: 2023-09-04 10:10:04
tags: 
- Web
categories:
- Web
---

- Debian服务端使用nvm安装<Node.js>
    1. 首先安装nvm，进入nvm的[github页面](https://github.com/nvm-sh/nvm)下载install.sh并使用WinSCP上传至服务器，然后bash运行；或使用cURL或者Wget命令([参考](https://github.com/nvm-sh/nvm/blob/master/README.md))
    ```bash
    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash

    wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.5/install.sh | bash
    ```

    2. 利用nvm安装nodejs
    ```bash
    # 查看可以安装独立的Node.js版本
    nvm ls-remote

    # 根据需要安装，若最新的LTS版本为v18.18.0则
    nvm install 18.18.0

    # 安装成功提示
    Checksums matched!
    Now using node v18.18.0 (npm v9.8.1)
    Creating default alias: default -> 18.18.0 (-> v18.18.0)
    ```

- 安装Twikoo server
```bash
npm i -g tkserver
```
- 根据需要配置环境变量，所有环境变量都是可选的([参考](https://twikoo.js.org/backend.html#%E7%A7%81%E6%9C%89%E9%83%A8%E7%BD%B2))

| 名称 | 描述 | 默认值 |
| :----- | :----- | :----- |
| ```MONGODB_URI```             | MongoDB 数据库连接字符串，不传则使用 lokijs                       | ```null```   |
| ```MONGO_URL```               | MongoDB 数据库连接字符串，不传则使用 lokijs                       | ```null```   |
| ```TWIKOO_DATA```             | lokijs 数据库存储路径                                            | ```./data``` |
| ```TWIKOO_PORT```             | 端口号                                                          | ```8080```   | 
| ```TWIKOO_THROTTLE```         | IP 请求限流，当同一 IP 短时间内请求次数超过阈值将对该 IP 返回错误    | ```250```    |
| ```TWIKOO_LOCALHOST_ONLY```   | 为```true```时只监听本地请求，使得 nginx 等服务器反代之后不暴露原始端口 | ```null``` |
| ```TWIKOO_LOG_LEVEL```        | 日志级别，支持 ```verbose``` / ```info``` / ```warn``` / ```error``` | ```info``` |
| ```TWIKOO_IP_HEADERSTWIKOO_IP_HEADERS``` | 在一些特殊情况下使用，如使用了CloudFlare CDN它会将请求IP写到请求头的```cf-connecting-ip```字段上，为了能够正确的获取请求 IP 你可以写成```['headers.cf-connecting-ip']``` | ```[]``` |

- 启动Twikoo server并访问```http://服务端IP:8080```测试([参考](https://www.drflower.top/posts/7a36ee43/#%E7%8E%AF%E5%A2%83))
```bash
tkserver # 启动服务
nohup tkserver >> tkserver.log 2>&1 & # 后台启动服务
kill $(ps -ef | grep tkserver | grep -v 'grep' | awk '{print $2}') # 停止服务
netstat -apn # 查看端口占用情况
```

- 申请ssl证书并在DNS服务商处添加```twikoo.sawubona.cn```的解析，然后配置nginx前置代理实现HTTPS访问
```nginx
server {
     listen 443 ssl; 
     server_name twikoo.sawubona.cn;
     ssl_certificate twikoo.sawubona.cn_bundle.crt; 
     ssl_certificate_key twikoo.sawubona.cn.key; 
     ssl_session_timeout 5m;
     ssl_protocols TLSv1.2 TLSv1.3; 
     ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; 
     ssl_prefer_server_ciphers on;
 
     location / {
     proxy_set_header Host $host;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header X-Forwarded-Proto $scheme;
     proxy_set_header REMOTE-HOST $remote_addr;
     proxy_pass  http://localhost:8080;
     }
}
server {
 listen 80;
 server_name twikoo.sawubona.cn;
 return 301 https://$host$request_uri;    
}
```

- 在博客配置文件中配置envld，然后在网站评论区设置管理登录密码。