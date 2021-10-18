# proxy_nginx

## 构建和调试

**构建镜像：**

基于`webdevops/php-nginx:7.4`；

```bash
# Build
cd /root/Git/proxy_nginx
docker build -t wdssmq/proxy_nginx .
```

**运行：**

```shell
# 复制一份生产环境配置
GIT_DIR=/root/Git/proxy_nginx
PRD_DIR=/home/www/proxy_nginx_prd
if [ ! -d $PRD_DIR ]; then
  cp -r $GIT_DIR $PRD_DIR
fi
```

```shell
# 调试运行
PRD_DIR=/home/www/proxy_nginx_prd
CONF_DIR=${PRD_DIR}/nginx
APP_DIR=${PRD_DIR}/app

CON_NAME=proxy_nginx
 
DEBUG=1
# 删除
docker rm --force $CON_NAME
if [ "$DEBUG" -eq "1" ]; then
  docker run --rm --name $CON_NAME \
    -p 80:80 \
    -p 443:443 \
    -v "${APP_DIR}":/app \
    -v "${CONF_DIR}":/opt/docker/etc/nginx \
    wdssmq/proxy_nginx
else
  docker run -d --name $CON_NAME \
    -p 80:80 \
    -p 443:443 \
    -v "${APP_DIR}":/app \
    -v "${CONF_DIR}":/opt/docker/etc/nginx \
    wdssmq/proxy_nginx
fi
# exit

# 查看日志
docker logs $CON_NAME

# 进入容器内
docker exec -it $CON_NAME /bin/bash

# 「调试」复制文件
# 在容器外
TEST_DIR=/home/www/test_proxy_nginx
if [ ! -d $TEST_DIR ]; then
  mkdir -p $TEST_DIR
fi
docker cp $CON_NAME:/opt/docker/etc/nginx "${TEST_DIR}/"
# docker cp $CON_NAME:/etc/nginx/nginx.conf "${TEST_DIR}/"

```

### 申请 ssl 证书

**使用 Docker 镜像申请证书：**

```shell
PRD_DIR=/home/www/proxy_nginx_prd
CER_DIR=${PRD_DIR}/letsencrypt
APP_DIR=${PRD_DIR}/app

docker run -it --rm --name certbot \
 -v "${CER_DIR}/etc:/etc/letsencrypt" \
 -v "${CER_DIR}/lib:/var/lib/letsencrypt" \
 -v "${CER_DIR}/log:/var/log/letsencrypt" \
 -v "${APP_DIR}:/var/www" \
 certbot/certbot certonly

# 复制进 nginx/ssl/
cp -r ${CER_DIR}/etc/archive ${PRD_DIR}/nginx/ssl/
cp -r ${CER_DIR}/etc/live ${PRD_DIR}/nginx/ssl/
```

**修改`${PRD_DIR}/nginx/vhost.ssl.conf`：**

```nginx
# ssl_certificate     /opt/docker/etc/nginx/ssl/server.crt;
# ssl_certificate_key /opt/docker/etc/nginx/ssl/server.key;
ssl_certificate     /opt/docker/etc/nginx/ssl/live/yourdomain.com/fullchain.pem;
ssl_certificate_key /opt/docker/etc/nginx/ssl/live/yourdomain.com/privkey.pem;
```

**重启镜像容器**

```shell
CON_NAME=proxy_nginx
docker restart $CON_NAME
docker logs $CON_NAME
```

**证书申请细节：**

- 如果 80 端口已经占用中，则选择「`webroot`」；
- 输入邮箱；
- 同意协议；
- 拒绝邮件推送；
- 输入要申请证书的域名；
- 输入「`webroot`」路径，此处为`/var/www`；

成功后会给出存放地址，实际路径则为上边映射的路径；

需要注意给出的路径是符号链接，实际文件在`……/etc/archive/域名/`下；


```shell
Saving debug log to /var/log/letsencrypt/letsencrypt.log

How would you like to authenticate with the ACME CA?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: Spin up a temporary webserver (standalone)
2: Place files in webroot directory (webroot)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 2
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): 12345@qq.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: N
Account registered.
Please enter the domain name(s) you would like on your certificate (comma and/or
space separated) (Enter 'c' to cancel): www.bing.com
Requesting a certificate for www.bing.com
Input the webroot for www.bing.com: (Enter 'c' to cancel): /var/www

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/www.bing.com/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/www.bing.com/privkey.pem
This certificate expires on 2022-01-16.
These files will be updated when the certificate renews.

NEXT STEPS:
- The certificate will need to be renewed before it expires.
- Certbot can automatically renew the certificate in the background,
  but you may need to take steps to enable that functionality.
- See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le

```