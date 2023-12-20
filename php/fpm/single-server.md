# PHP-FPM LAMP STACK

- variables
    ```bash
    PHP_VERSION=7.2
    ```
- update the OS
    ```bash
    apt update && apt full-upgrade -y
    ```

- Installing an Apache
    ```bash 
    apt install apache2 && ufw allow in "Apache" && echo "Check it in your browser: $(curl -s curl http://icanhazip.com)"
    ```

## Adding some Repo for installing the php-fpm
- Ondrej PPA repository
```bash
apt-get install software-properties-common  && add-apt-repository -y ppa:ondrej/php && apt update
```

- Installation of php-fpm along some PHP Extensions
```bash
apt install php${PHP_VERSION}-{fpm,common,zip,curl,xml,xmlrpc,json,mysql,pdo,gd,imagick,ldap,imap,mbstring,intl,cli,tidy,bcmath,opcache}
```

- Ensure version
```bash
php${PHP_VERSION} -v
```

- For PHP Info Page
```bash

```
