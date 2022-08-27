# Configuring TT-RSS on OpenBSD
#### Install required packages:
```
pkg_add php php-pdo_pgsql php-pgsql php-curl php-intl postgresql-server git gnupg unzip
```

#### Edit /etc/php-XX.ini and enable these:
```
extension=pdo_pgsql.so
extension=pgsql.so
extension=intl.so
extension=curl.so
zend_extension=opcache.so
```

#### Using Let's Encrypt for SSL certification 
#### edit /etc/acme-client.conf:
```
authority letsencrypt {
        api url "https://acme-v02.api.letsencrypt.org/directory"
        account key "/etc/acme/letsencrypt-privkey.pem"
}
authority letsencrypt-staging {
        api url "https://acme-staging.api.letsencrypt.org/directory"
        account key "/etc/acme/letsencrypt-staging-privkey.pem"
}
domain domain.tld {
        domain key "/etc/ssl/private/domain.tld.key"
        domain certificate "/etc/ssl/domain.tld.crt"
        domain full chain certificate "/etc/ssl/domain.tld_fullchain.pem"
        sign with letsencrypt
}
```

#### Fetch the certificates:
```
acme-client -v domain.tld
```

#### Obtain the OCSP stapling file:
```
ocspcheck -N -o /etc/ssl/domain.tld.ocsp.pem /etc/ssl/domain.tld_fullchain.pem
```

Or using a self-signed cert instead, ie:
#### Using a self-signed cert (10 years):
```
openssl genrsa -out /etc/ssl/private/server.key
openssl req -new -x509 -key /etc/ssl/private/server.key -out /etc/ssl/server.crt -days 3365
```

#### Edit crontab to renew certs and obtain stapling file:
```
doas crontab -e
0 0 * * * acme-client domain.tld && rcctl reload httpd
0 * * * * ocspcheck -N -o /etc/ssl/domain.tld.ocsp.pem /etc/ssl/domain.tld_fullchain.pem && rcctl reload httpd
```

#### Create or edit /etc/httpd.conf, ie:
```
ext_addr="*"
domain="domain.tld"
prefork 3

server $domain {
        listen on $ext_addr port 80
        block return 301 "https://$SERVER_NAME$REQUEST_URI"
}

server $domain {
        listen on $ext_addr tls port 443

        tls {
                certificate "/etc/ssl/domain.tld_fullchain.pem"
                key "/etc/ssl/private/domain.tld.key"
                ocsp "/etc/ssl/domain.tld.ocsp.pem"
        }

        directory {
                index "index.php"
        }

        location "*.php*" {
                fastcgi socket "/run/php-fpm.sock"
        }

        location "/tt-rss" {
                root "/htdocs/tt-rss"
        }
}

# Include MIME types instead of the built-in ones
types {
        include "/usr/share/misc/mime.types"
}
```

#### Permit DNS and hostfile resolution, within the chroot:
```
mkdir -p /var/www/etc
sh -c "grep ^nameserver /etc/resolv.conf > /var/www/etc/resolv.conf"
touch /var/www/etc/hosts # optional
```

#### Enable services (not starting):
```
rcctl enable httpd postgresql phpXX_fpm
```

#### Initialize postgresql:
```
su - _postgresql
mkdir /var/postgresql/data
initdb -D /var/postgresql/data -U postgres -A scram-sha-256 -E UTF8 -W
logout
```

#### Start postgresql and add a user:
```
rcctl start postgresql
su - _postgresql
psql -U postgres
        postgres=# CREATE USER "tt-rss" WITH PASSWORD 'YourPasswordHere';
        postgres=# CREATE DATABASE ttrssdb WITH OWNER "tt-rss";
        postgres=# GRANT ALL PRIVILEGES ON DATABASE ttrssdb to “tt-rss”;
        postgres=# \quit
```

#### Edit: /etc/php-X.X.ini and set some values, ie:
```
memory_limit = 256M
max_input_time = 180
upload_max_filesize = 512M
post_max_size = 32M
opcache.enable = 1
opcache.enable_cli = 1
opcache.memory_consumption = 128
opcache.interned_strings_buffer = 8
opcache.max_accelerated_files = 10000
opcache.revalidate_freq = 1
opcache.save_comments = 1
```

#### Clone tt-rss:
```
git clone --depth=1 https://tt-rss.org/git/tt-rss.git /var/www/htdocs/tt-rss
chown -R www:www /var/www/htdocs/tt-rss
```

#### Set up the SSL certs:
```
cp -R /etc/ssl/ /var/www/etc/
```

#### Periodically check and update feeds in a crontab:
```
crontab -u www -e
MAILTO=””
*/30 * * * * /usr/local/bin/php-8.0 /var/www/htdocs/tt-rss/update.php --feeds --quiet
```

#### Symlink php (or edit php scripts):
```
ln -s /usr/local/bin/php-X.X /usr/bin/php
ln -s /usr/local/bin/php-X.X /usr/local/bin/php
```

#### Global Configuration:
```
cd /var/www/htdocs/tt-rss/
cp config.php-dist config.php
```

#### Edit config.php (optional?):
```
putenv('TTRSS_DB_HOST=localhost’);
putenv('TTRSS_DB_NAME=ttrssdb’);
putenv('TTRSS_DB_USER=tt-rss’);
putenv('TTRSS_DB_PASS=YourPasswordHere');
putenv('TTRSS_SELF_URL_PATH=https://domain.tld/tt-rss');
```

#### Manually update schema (optional):
```
doas -u www php /var/www/htdocs/tt-rss/update.php --update-schema
```

* Connect to your server: https://domain.tld/tt-rss/
* Default user/pass: admin/password
* Change admin password!

#### Download themes:
```
ftp https://github.com/levito/tt-rss-feedly-theme/archive/master.zip
unzip master.zip
cd tt-rss-feedly-theme-master
cp -r feedly* [TT-RSS_Home]/themes.local

* In TT-RSS preferences, select the feedly-theme.
```

#### Notes / Resources:
```
https://tt-rss.org/wiki/InstallationNotesHost
https://github.com/levito/tt-rss-feedly-theme

* tt-rss version is not displaying. Reason / solution to be determined.
```
