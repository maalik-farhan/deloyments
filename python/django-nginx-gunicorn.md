# Django Project Setup

-   Set Time and TimeZone

    ```bash
    timedatectl set-timezone Asia/Karachi &&
    timedatectl set-local-rtc false &&
    timedatectl set-ntp false &&
    timedatectl set-ntp true
    ```

-   Update the OS
    ```bash
    apt update && apt full-upgrade -y
    ```

-   Reboot the OS
    ```bash
    reboot
    ```

-   Create variables
    ```bash
    CUS_LOGS=/var/log/custom_logs && APP_NAME='farhan.test.com' && APP_DIR='/var/python/django/'$APP_NAME && PROJECT_NAME='farhan_test'
    ```

## Logging

```bash
mkdir -p ${CUS_LOGS} &&
echo ${CUS_LOGS}'/*.log
{
daily
missingok
rotate 14
compress
delaycompress
notifempty
create 640 root adm
sharedscripts
copytruncate
}
' | tee /etc/logrotate.d/lr_custom >/dev/null && logrotate /etc/logrotate.d/lr_custom;
```

## Install Nginx and other required packages.

- `default-libmysqlclient-dev` needs research.
```bash
apt install build-essential {libssl,libffi}-dev nginx python3 python3-{pip,dev,venv} default-libmysqlclient-dev zip unzip whois
```

## Services Setup

-   Add scripts to Reload and Restart **Nginx** and **a1.trackmypackge.net**.

    ```bash
    echo '#!/bin/bash
    systemctl reload '${APP_NAME}'.service; systemctl status '${APP_NAME}'.service --no-pager;
    printf "%*s\n" "${COLUMNS:-$(tput cols)}" "" | tr " " -;
    systemctl reload nginx.service; systemctl status nginx.service --no-pager;
    ' | tee /usr/local/bin/reload-server.sh >/dev/null &&
    chmod +x /usr/local/bin/reload-server.sh;\
    echo '#!/bin/bash
    systemctl restart '${APP_NAME}'.service; systemctl status '${APP_NAME}'.service --no-pager;
    printf "%*s\n" "${COLUMNS:-$(tput cols)}" "" | tr " " -;
    systemctl restart nginx.service; systemctl status nginx.service --no-pager;
    ' | tee /usr/local/bin/restart-server.sh >/dev/null &&
    chmod +x /usr/local/bin/restart-server.sh
    ```

    -   Confirm changes.

        ```bash
        nano /usr/local/bin/re{load,start}-server.sh
        ```

## Prepare and Activate Python Virtual Environment

```bash
mkdir -p ${APP_DIR} && mkdir -p ${APP_DIR}/log && cd ${APP_DIR} && python3 -m venv env
```

-   To activate env
    ```bash
    source env/bin/activate
    ```
-   Install necessary packages
    ```
    pip install django gunicorn
    ```
-   Make a project for django
    ```
    django-admin startproject ${PROJECT_NAME} .
    ```
-   Make sure in setting.py in project folder have edit allow_domains

-   Upload files.

## Gunicorn Sockets

-   To make a socket file
    ```bash
    echo '
    [Unit]
    Description=Project Description

    [Socket]
    ListenStream='${APP_DIR}'/'${APP_NAME}'.sock

    [Install]
    WantedBy=sockets.target
    ' | tee /etc/systemd/system/${APP_NAME}.socket >/dev/null
    ```

-   Make a service file
    ```bash
    echo '
    [Unit]
    Description=gunicorn daemon
    Requires='${APP_NAME}'.socket
    After=network.target

    [Service]
    User=www-data
    Group=www-data
    WorkingDirectory='${APP_DIR}'
    ExecStart='${APP_DIR}'/env/bin/gunicorn --workers 3 --bind unix:'${APP_DIR}'/'${APP_NAME}'.sock '${PROJECT_NAME}'.wsgi:application

    [Install]
    WantedBy=multi-user.target
    ' | tee /etc/systemd/system/${APP_NAME}.service >/dev/null
    ```

    -   Confirm changes.

        ```bash
        nano /etc/systemd/system/${APP_NAME}.service
        ```

-   Now start and enable gunicorn socket
    ```bash
    systemctl start ${APP_NAME}.socket
    systemctl enable ${APP_NAME}.socket
    ```

## Nginx Configuration

-   Turn-Off **Nginx** Signatures.

    ```bash
    sed -i'.bkp' -e 's/# server_tokens off;/server_tokens off;/;' /etc/nginx/nginx.conf
    ```

    -   Confirm changes.

        ```bash
        nano /etc/nginx/nginx.conf
        ```

-   Change **Nginx** directory structure. (As needed)

    ```bash
    mv /etc/nginx/conf.d /etc/nginx/conf-available && mkdir /etc/nginx/conf-enabled && ln -s /etc/nginx/conf-available/* /etc/nginx/conf-enabled/ && ln -s /etc/nginx/conf-enabled /etc/nginx/conf.d
    ```

    -   Confirm changes.

        ```bash
        ll /etc/nginx /etc/nginx/sites-available /etc/nginx/sites-enabled /etc/nginx/conf.d /etc/nginx/conf-available /etc/nginx/conf-enabled
        ```

-   Add configuration for **Nginx**'s _ngx_http_realip_module_ module (in case of CloudFlare)

    ```bash
    echo -e "real_ip_header CF-Connecting-IP;

    # https://www.cloudflare.com/ips-v4
    $(for ip in $(curl -sL https://www.cloudflare.com/ips-v4); do echo "set_real_ip_from ${ip};"; done;)

    # https://www.cloudflare.com/ips-v6
    $(for ip in $(curl -sL https://www.cloudflare.com/ips-v6); do echo "set_real_ip_from ${ip};"; done;)

    # Add IP of Load Balancer
    #set_real_ip_from 0.0.0.0/32;
    " | tee /etc/nginx/conf-available/10-remoteip.conf >/dev/null && ln -s /etc/nginx/conf-available/10-remoteip.conf /etc/nginx/conf-enabled/
    ```

    -   Confirm changes.

        ```bash
        nano /etc/nginx/conf.d/10-remoteip.conf
        ```

-   As **GZip** compression is _on_, change format for **combined** _Log Format_.

    ```bash
    echo "log_format compression '\$remote_addr - \$remote_user [\$time_local] \"\$request\" \$status \$body_bytes_sent \"\$http_referer\" \"\$http_user_agent\" \"\$gzip_ratio\"';" | tee /etc/nginx/conf-available/20-log_formats.conf >/dev/null && ln -s /etc/nginx/conf-available/20-log_formats.conf /etc/nginx/conf-enabled/
    ```

    -   Confirm changes.

        ```bash
        nano /etc/nginx/conf.d/20-log_formats.conf
        ```
-   Change **Nginx**'s default virtual host configuration.

    ```bash
    rm /etc/nginx/sites-enabled/default && mv /etc/nginx/sites-available/default /etc/nginx/sites-available/000-default.conf.bkp &&
    echo '# Default server configuration
    #
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        server_name _;
        access_log '${CUS_LOGS}'/000-access.log compression;
        error_log '${CUS_LOGS}'/000-error.log;

        root /var/www/html;

        allow 127.0.0.;
        allow 137.59.225.112/32;
        allow 103.178.216.48/29;
        allow 103.178.217.96/28;
        allow 68.232.175.56/32;
        allow 2001:19f0:5:4a8:5400:2ff:fec0:6bed/128;
        deny all;

        location / {
            try_files $uri $uri/ =404;
        }

        location = /health_check.html {
            allow all;
        }
    }
    ' | tee /etc/nginx/sites-available/000-default.conf >/dev/null
    ```

    -   Confirm changes.

        ```bash
        nano /etc/nginx/sites-available/000-default.conf
        ```

    -   Enable default site.

        ```bash
        ln -s /etc/nginx/sites-available/000-default.conf /etc/nginx/sites-enabled/
        ```

-   Add file for **Nginx** default _virtual host_ and _health_check_..

    ```bash
    echo 'Farhan Scratch' | tee /var/www/html/index.html >/dev/null;\
    echo 'Server OK' | tee /var/www/html/health_check.html >/dev/null
    ```

-   Change ownership of newly created files.

    ```bash
    chown -R www-data:www-data /var/www/
    ```

-   Add virtual host.

    ```bash
    echo "
    server {
        listen 80;
        listen [::]:80;
        server_name "${APP_NAME}";
        access_log "${CUS_LOGS}"/"${APP_NAME}"-access.log compression;
        error_log "${CUS_LOGS}"/"${APP_NAME}"-error.log;

        client_max_body_size 10M;

        location / {
            include proxy_params;
            proxy_read_timeout 180;
            proxy_pass http://unix:"${APP_DIR}"/"${APP_NAME}".sock;
        }
    }
    " | tee /etc/nginx/sites-available/${APP_NAME}.conf >>/dev/null
    ```

    -   Confirm changes.

        ```bash
        nano /etc/nginx/sites-available/${APP_NAME}.conf
        ```

-   Enable virtual host.

    ```bash
    ln -s /etc/nginx/sites-available/${APP_NAME}.conf /etc/nginx/sites-enabled/
    ```

-   Test **Nginx** Config.

    ```bash
    nginx -t
    ```

-   Restart required services.

    ```bash
    restart-server.sh
    ```

# References

- https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-20-04
- https://www.digitalocean.com/community/tutorials/how-to-install-python-3-and-set-up-a-programming-environment-on-an-ubuntu-20-04-server
- https://www.uvicorn.org/deployment/#running-behind-nginx
- https://www.vultr.com/docs/how-to-deploy-fastapi-applications-with-gunicorn-and-nginx-on-ubuntu-20-04/