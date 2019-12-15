# Website
[ip-arxiv.org](http://ip-arxiv.org) is currently hosted on a FreeNAS server jail infront of a nginx reverse proxy with a cron job that periodically updates the DNS A entry of ip-arxiv.org to my pubic ip address.

## Router
Open port 80 and forward all requests to nginx reverse proxy, which has the local ip `192.168.1.3`.

## Reverse Proxy
The reverse proxy jail contains an nginx webserver and an dns-upater bash script, which gets regularly executed in a FreeNAS cron job.

### Nginx
Every request from [ip-arxiv.org](http://ip-arxiv.org) shall be redirected to the wordpress server, which has the local ip `192.168.1.6`
``` shell
#/usr/local/etc/nginx/nginx.conf
worker_processes  1;

events {
    worker_connections  1024;
}


http {
    server {
        listen       80;
        server_name  ip-arxiv.org;

        location / {
		proxy_pass http://192.168.1.6:80/;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
		proxy_buffering off;
		client_max_body_size 50M;
        }
    }
}
```

### DNS-Updater
``` shell
#/usr/local/etc/dns-updater/dns-updater.sh
#!/usr/local/bin/bash
DOMAIN="ip-arxiv.org"
ROUTE53_HOSTED_ZONE_ID=Z1X4K2OKA7LS6T

DOMAIN_A_RECORD=$(host $DOMAIN | awk ' { print $4 }')
LAN_PUBLIC_IP=$(curl http://ifconfig.co)

#echo $LAN_PUBLIC_IP
#echo $DOMAIN_A_RECORD

if [ "$LAN_PUBLIC_IP" != "$DOMAIN_A_RECORD" ]
then
cat > /tmp/dns-update.json << __EOF__
  {
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "${DOMAIN}",
          "Type": "A",
          "TTL": 600,
          "ResourceRecords": [
            {
              "Value": "${LAN_PUBLIC_IP}"
            }
          ]
        }
      }
    ]
  }
__EOF__
aws route53 change-resource-record-sets \
	--hosted-zone-id $ROUTE53_HOSTED_ZONE_ID \
	--change-batch file:///tmp/dns-update.json
echo "DNS updated"
echo "updated" >> /usr/local/etc/dns-updater/log.txt
else
echo "Nothing to do"
fi
```
We have to make the bash script executable and symlink it:
``` shell
chmod +x ./dns-updater.sh
ln -s /usr/local/etc/dns-updater/dns-updater.sh /usr/local/bin/dns-updater
```
Cron jobs in FreeNAS can be found under: `Tasks -> Cron Jobs`. The command is `jexec ioc-ReverseProxy dns-updater`. Minute, Hour, Day of Month should be `*`.

## FAMP (FreeBSD + Apache + MySQL + PHP) for Wordpress
Install dependencies
``` shell
pkg update && pkg upgrade
pkg install apache24 mod_php73 php73-mysqli mysql56-server php73-xml php73-hash php73-gd php73-curl php73-tokenizer php73-zlib php73-zip php73-extensions
```
(Small remark: if we would install mysql8 we also would make sure that the user uses a different authorization mechanism than the standard one. Otherways php mysql_connect cannot connect properly.)

Configure PHP Apache
``` shell
#/usr/local/etc/apache24/Includes/php.conf
<IfModule dir_module>
    DirectoryIndex index.php index.html

    <FilesMatch "\.php$">
        SetHandler application/x-httpd-php
    </FilesMatch>
    <FilesMatch "\.phps$">
        SetHandler application/x-httpd-php-source
    </FilesMatch>
</IfModule>
```

``` shell
cd /usr/local/etc
cp php.ini-production php.ini
vim php.ini
# php.ini
upload_max_filesize = 32M
```

Autostart Apache and MySQL
``` shell
sysrc apache24_enable=YES
sysrc mysql_enable=YES

service apache24 start
service mysql-server start
```

Configure MySQL
``` shell
mysql_secure_installation # choose secure passwort, accept rest
mysql -u root -p
> CREATE DATABASE wordpress;
> CREATE USER wp_user@localhost IDENTIFIED BY 'wp_pass';
> GRANT ALL PRIVILEGES ON wordpress.* TO wp_user@localhost;
> FLUSH PRIVILEGES;
> exit
```

Install Wordpress
``` shell
cd ~
fetch https://wordpress.org/latest.tar.gz
tar -xzvf latest.tar.gz
mv wordpress/* /usr/local/www/apache24/data/
chown -R www:www /usr/local/www/apache24/data/*
cd /usr/local/www/apache24/data
cp wp-config-sample.php wp-config.php
rm index.html
vi wp-config.php
```

Configure Wordpress
``` php
define( 'DB_Name, 'wordpress' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'wp_pass' );
define( 'DB_HOST', 'localhost' );

define('AUTH_KEY',         'aZ{qTC%Ur.4mSCkj[@r&>DyI.+.{yPh(T7O~CiKvV+ubm*Q!U$C<wImW?=yR7/s!');
define('SECURE_AUTH_KEY',  '/.Xtpc_C%FGl@}8_x2%Hc TG<`mY;,s{kBc!|b/+iukqkT4uE.w|>-JKzoT`Q`lM');
define('LOGGED_IN_KEY',    'fdh:!v[-9p8Mau&s(G-=zKD8H)*[_XN3@`jCup`HDWr[fhlSiV%{-Uxl9#*M&-_4');
define('NONCE_KEY',        'tMDfKZ7@+`k(EhX^?V6tZ%*W0]d_wR_6)IIhk0vQHeiJ-RgpZe)-Y:T]yJtO8+xU');
define('AUTH_SALT',        'DA]f3FK**,)$+Z/z`JlTr<1)@cFwvM,*z(OWC$D?D<;R<h+TtinFmA(X,!j]+dS,');
define('SECURE_AUTH_SALT', '{|P}Ll+f))Q;AoL#xd~x|^;`nLs7aly(d*X$CQp`5m!L|P9a|*vAU1?&lL#*|Vl/');
define('LOGGED_IN_SALT',   '# S-IQtIB]b1+Z@*@I&sc@NoqhV]36Y-e2)itRT`C]1]H`Us0G}>(;C+|LEq?_oY');
define('NONCE_SALT',       'wJB:54=VmO[aga}uI<.v`A|P!Vc-HcC,}=;DEXf8wAPr%6woWL:fhgik@D|!HTtj');
```




