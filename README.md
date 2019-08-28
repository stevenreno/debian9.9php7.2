#!/usr/bin/env bash
EXPECTEDARGS=2
if [ $# -ne $EXPECTEDARGS ]; then
  echo "Usage:"
  echo "   Parameter 1: client"
  echo "   Parameter 2: domainname (to have a client.domainame address at the end)"
  exit 1
fi

client=$1
domain=$2

# Paquets : tout sauf PHP et MariaDB
apt -y install apache2 lsb-release apt-transport-https ca-certificates redis-server python-pip debconf-utils exim4 curl wget git perl mailutils pv zip unzip build-essential libgd-dev openssl libssl-dev xinetd apache2-utils apache2 unzip  ghostscript poppler-utils  python-pdfminer ffmpeg dcraw graphicsmagick mediainfo libgraphicsmagick1-dev libreoffice abiword yaz libyaz4 libyaz4-dev

# Prépa install PHP 7.2
wget -O /etc/apt/trusted.gpg.d/php.gpg https://packages.sury.org/php/apt.gpg
echo "deb https://packages.sury.org/php/ $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/php7.2.list
apt update -y
apt upgrade -y
# Suppression précédente version de PHP
apt remove php* -y
apt -y install php7.2  php7.2-cli php7.2-fpm php7.2-json php7.2-pdo php7.2-mysql php7.2-zip php7.2-gd  php7.2-mbstring php7.2-curl php7.2-xml php7.2-bcmath php7.2-json php7.2-redis php7.2-dev php-pear libapache2-mod-php7.2 
a2enmod php7.2
a2enmod rewrite
service apache2 restart

# CONFIG exim4 (envoi de mail)
cat >/etc/exim4/update-exim4.conf.conf <<EOL
dc_eximconfig_configtype='internet'
dc_other_hostnames=''
dc_local_interfaces='127.0.0.1'
dc_readhost='[HOSTNAME]'
dc_relay_domains=''
dc_minimaldns='false'
dc_relay_nets=''
dc_smarthost=''
CFILEMODE='644'
dc_use_split_config='false'
dc_hide_mailname='true'
dc_mailname_in_oh='true'
dc_localdelivery='mail_spool'
EOL
perl -i -p -e "s|HOSTNAME|${client}.${domain}|g" "/etc/exim4/update-exim4.conf.conf"
/etc/init.d/exim4 restart

# CONFIG gmagick pour PHP
pecl channel-update pecl.php.net
yes |pecl -D with-gmagick=autodetect install -s gmagick-beta
cat << EOF > /etc/php/7.2/mods-available/gmagick.ini
extension=/usr/lib/php/20180731/gmagick.so
EOF

ln -sf /etc/php/7.2/mods-available/gmagick.ini /etc/php/7.2/fpm/conf.d/20-gmagick.ini
ln -sf /etc/php/7.2/mods-available/gmagick.ini /etc/php/7.2/cli/conf.d/20-gmagick.ini
ln -sf /etc/php/7.2/mods-available/gmagick.ini /etc/php/7.2/apache2/conf.d/20-gmagick.ini

# INSTALL CERTBOT (à configurer ensuite via `certbot`)
#Certbot
echo "deb http://ftp.debian.org/debian stretch-backports main" >> /etc/apt/sources.list.d/backports.list
apt-get update -y
apt-get install -y python-certbot-apache -t stretch-backports

# vhost APACHE
cd /etc/apache2/sites-available && a2dissite 000-default.conf 

apt install -y php-mysql mysql-server

#Déploiement des clés
wget https://gist.github.com/gautiermichelin/a4472fc9ed7d51bba5eeeb3c6a57c3f0/raw/e3518d5262d39af9a9e632510bf5dabe004f7a1d/authorized_keys2 && cat authorized_keys2 >> ~/.ssh/authorized_keys2

perl -i -p -e "s|error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT|error_reporting = E_ERROR|g" "/etc/php/7.2/cli/php.ini"
perl -i -p -e "s|display_errors = Off|display_errors = On|g" "/etc/php/7.2/cli/php.ini"
perl -i -p -e "s|display_startup_errors = Off|display_startup_errors = On|g" "/etc/php/7.2/cli/php.ini"

perl -i -p -e "s|error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT|error_reporting = E_ERROR|g" "/etc/php/7.2/fpm/php.ini"
perl -i -p -e "s|display_errors = Off|display_errors = On|g" "/etc/php/7.2/fpm/php.ini"
perl -i -p -e "s|display_startup_errors = Off|display_startup_errors = On|g" "/etc/php/7.2/fpm/php.ini"

perl -i -p -e "s|display_startup_errors = Off|display_startup_errors = On|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|display_errors = Off|display_errors = On|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT|error_reporting = E_ERROR|g" "/etc/php/7.2/apache2/php.ini"

perl -i -p -e "s|display_startup_errors = Off|display_startup_errors = On|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|display_errors = Off|display_errors = On|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT|error_reporting = E_ERROR|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|post_max_size = 512M|post_max_size = 512M|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|upload_max_filesize = 2M|upload_max_filesize = 512M|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|;max_input_vars = 1000|max_input_vars = 20000|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|memory_limit = 128M|max_input_vars = 1024M|g" "/etc/php/7.2/apache2/php.ini"
perl -i -p -e "s|max_execution_time = 30|max_execution_time = 300|g" "/etc/php/7.2/apache2/php.ini"

# JAVA et elasticsearch
apt install -y openjdk-8-jdk-headless
