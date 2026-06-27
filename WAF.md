# 🛡️ Lesson 3: WAF - Web Application Firewall

## 1. DVWA Quraşdırılması (Client VM)

### a. Tələb olunan paketləri quraşdırın

```bash
sudo apt install apache2 mysql-server php php-mysql php-gd libapache2-mod-php
b. DVWA source kodunu klonlayın
bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
c. MySQL verilənlər bazasını qurun
bash
sudo mysql -u root -p
# (şifrə yoxdur, sadəcə Enter)
sql
CREATE DATABASE dvwa;
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
FLUSH PRIVILEGES;
EXIT;
d. DVWA config faylını redaktə edin
bash
sudo cp /var/www/html/DVWA/config/config.inc.php.dist /var/www/html/DVWA/config/config.inc.php
sudo nano /var/www/html/DVWA/config/config.inc.php
e. Apache-nı restart edin
bash
sudo systemctl restart apache2
f. DVWA-ya daxil olun
Brauzerdə: http://<CLIENT_VM_IP>/DVWA

Giriş məlumatları:

İlk login: root / P@ssw4rd

"Create/Reset Database" düyməsinə basın

2-3 dəqiqə gözləyin

İkinci login: admin / password

2. NGINX + ModSecurity Quraşdırılması (WAF VM)
a. İlkin paketləri quraşdırın
bash
sudo apt install -y git autoconf automake libtool libpcre3 libpcre3-dev \
libxml2 libxml2-dev libyajl-dev pkg-config libcurl4-openssl-dev libgeoip-dev \
libmaxminddb-dev doxygen liblua5.3-dev libssl-dev libz-dev g++ flex bison \
curl apache2-dev
b. ModSecurity library-ni build edin
bash
cd /opt
sudo git clone --depth 1 -b v3/master https://github.com/SpiderLabs/ModSecurity
cd ModSecurity
sudo git submodule init
sudo git submodule update
sudo ./build.sh
sudo ./configure
sudo make install
c. ModSecurity-nginx connector-u klonlayın
bash
cd /opt
sudo git clone https://github.com/SpiderLabs/ModSecurity-nginx.git
d. Nginx-i ModSecurity ilə compile edin
bash
cd /opt
sudo wget http://nginx.org/download/nginx-1.24.0.tar.gz
sudo tar -xvzf nginx-1.24.0.tar.gz
cd nginx-1.24.0
e. Nginx-i konfiqurasiya edin
bash
./configure --add-dynamic-module=/opt/ModSecurity-nginx
f. Nginx-i quraşdırın
bash
make
sudo make install
📝 Qeydlər
DB bağlantısında problem olarsa, localhost yerinə 127.0.0.1 yazın

DVWA üçün MariaDB istifadə edirsinizsə, root istifadə edə bilməzsiniz, ayrıca istifadəçi yaradın
