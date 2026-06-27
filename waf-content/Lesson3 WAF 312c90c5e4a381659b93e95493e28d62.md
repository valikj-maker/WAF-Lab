# Lesson3: WAF

## Install DVWA app on 1 VM (client VM)

1. Install required softwares

```bash
sudo apt install apache2 mysql-server php php-mysqli php-gd libapache2-mod-php git -y
```

b.  Clone DVWA source code

```bash
cd /var/www/html
sudo git clone https://github.com/digininja/DVWA.git
```

d. Set up DB

```bash
sudo mysql -u root -p //(no password. just enter. NOTE: run it with sudo)
CREATE DATABASE dvwa;
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
FLUSH PRIVILEGES;

```

e. Edit DVWA config file for DB connection

```bash
sudo cp /var/www/html/DVWA/config/config.inc.php.dist /var/www/html/DVWA/config/config.inc.php
sudo nano /var/www/html/DVWA/config/config.inc.php
```

![image.png](Lesson3%20WAF/image.png)

f. restart apache; login via: http://IP/DVWA:

- first login root:P@ssw0rd
- Click “create/reset database”
- If error occur, re open [http://IP/DVWA](http://IP/DVWA) after 2-3 minutes
- second login admin:password

## Install NGINX (waf VM)

1. Install NGINX and Modsecurity
    1. Install prerequisites v1
        
        ```bash
        sudo apt install \
            git \
            autoconf \
            automake \
            libtool \
            libpcre3 \
            libpcre3-dev \
            libxml2 \
            libxml2-dev \
            libyajl-dev \
            pkgconf \
            libcurl4-openssl-dev \
            libgeoip-dev \
            libmaxminddb-dev \
            doxygen \
            liblua5.3-dev \
            libssl-dev \
            libz-dev \
            -y
        
        ```
        
    2. Install prerequisites v2
        
        ```bash
        sudo apt install g++ flex bison curl apache2-dev doxygen libyajl-dev ssdeep liblua5.2-dev libgeoip-dev libtool dh-autoreconf libcurl4-gnutls-dev libxml2 libxml2-dev git liblmdb-dev libpkgconf3 lmdb-doc pkgconf zlib1g-dev libssl-dev -y
        ```
        
    3. Build and Install ModSecurity library
        
        ```bash
        cd /opt
        sudo git clone --depth 1 -b v3/master https://github.com/SpiderLabs/ModSecurity
        cd ModSecurity
        sudo git submodule init
        sudo git submodule update
        sudo ./build.sh
        sudo ./configure
        sudo make
        sudo make install
        
        ```
        
    4. Build ModSecurity-nginx Connector
        
        ```bash
        cd /opt
        sudo git clone https://github.com/SpiderLabs/ModSecurity-nginx.git
        ```
        
    5. Download and Compile Nginx with ModSecurity Module
        
        ```bash
        cd /opt
        sudo wget http://nginx.org/download/nginx-1.24.0.tar.gz
        sudo tar -xvzf nginx-1.24.0.tar.gz
        cd nginx-1.24.0
        ```
        
    6. Configure Nginx with dynamic module
        
        ```bash
        cd nginx-1.24.0
        sudo ./configure --with-compat --add-dynamic-module=../ModSecurity-nginx
        sudo make modules
        ```
        
    7. Copy module to nginx module directory (warning, do this step after installing nginx. Step e and f)
        
        ```bash
        sudo cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules/
        ```
        
    
    e. Install Nginx
    
    ```bash
    sudo apt install nginx -y
    ```
    
    f.  Create modules folder and add module to this folder
    
    ```bash
    sudo mkdir -p /etc/nginx/modules
    sudo cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules/
    ```
    
    g. Enable ModSecurity Module in Nginx (/etc/nginx/nginx.conf)
    
    ```bash
    user www-data;
    worker_processes auto;
    pid /run/nginx.pid;
    error_log /var/log/nginx/error.log;
    include /etc/nginx/modules-enabled/*.conf;
    
    load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;
    
    events {
            worker_connections 768;
            # multi_accept on;
    }
    
    http {
    
            ##
            # Basic Settings
            ##
    .........
    .........
    .........
    ```
    
    h. Create modsecurity config directory
    
    ```bash
    sudo mkdir /etc/nginx/modsec
    cd /etc/nginx/modsec
    ```
    
    1. Download modsecrity.conf
        
        ```bash
        sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/v3/master/modsecurity.conf-recommended
        sudo mv modsecurity.conf-recommended modsecurity.conf
        sudo nano modsecurity.conf
        ```
        
    
    j.  In modsecurity.conf file set *SecRuleEngine On*
    
    k.  Download unicode mapping file (inside /etc/nginx/modse/)
    
    ```bash
    sudo wget https://raw.githubusercontent.com/SpiderLabs/ModSecurity/master/unicode.mapping
    ```
    
    l. Remove *default* config file inside sites-available and create reverse proxy config file in /etc/nginx/sites-available/dvwa.conf
    
    ```bash
    server {
        listen 80;
        server_name _;
    
        modsecurity on;
        modsecurity_rules_file /etc/nginx/modsec/modsecurity.conf;
    
        location /DVWA/ {
            proxy_pass http://192.168.11.137:80/DVWA/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
    
            proxy_redirect off;
        }
    }
    ```
    
    m. Enable link the site
    
    ```bash
    rm /etc/nginx/sites-enable/default
    sudo ln -s /etc/nginx/sites-available/dvwa.conf /etc/nginx/sites-enabled/
    ```
    
    n. Test and restart nginx
    
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```
    
    o. Test if reverse proxy working or not
    
    p. Add this simple rule to /etc/nginx/modsec/modsecurity.conf
    
    ```bash
    SecRule ARGS "xss-test" "id:1234,phase:2,deny,log,status:403,msg:'XSS test blocked'"
    sudo systemctl restart nginx
    ```
    
    q. Restart nginx and check with this line
    
    ```bash
    curl "http://your-nginx-ip/DVWA/?test=xss-test"
    ```
    

1. Install OWASP Core Rule Set (CRS)
    1. 
        
        ```bash
        cd /etc/nginx/modsec
        sudo git clone https://github.com/coreruleset/coreruleset.git
        sudo cp coreruleset/crs-setup.conf.example coreruleset/crs-setup.conf
        ```
        
    2. Add these lines bottom of /etc/nginx/modsec/modsecurity.conf file 
        
        ```bash
        Include /etc/nginx/modsec/coreruleset/crs-setup.conf
        Include /etc/nginx/modsec/coreruleset/rules/*.conf
        ```
        
    3. Test DVWA again

### Test Reverse proxy with this config

```bash
server {
    listen 80;
    server_name dvwa.com;

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/modsecurity.conf;

    location /DVWA/ {
        proxy_pass http://192.168.0.108:80/DVWA/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_redirect off;
    }
}

server {
    listen 80;
    server_name dvwa2.com;

    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/modsecurity.conf;

    location /DVWA/ {
        proxy_pass http://192.168.0.16:80;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_redirect off;
    }
}

```

### Hide nginx servers version with “server_tokens” parameter

### Setup custom nginx page for block pages

1. Create custom html file
    
    ```bash
    sudo nano /usr/share/nginx/html/security-blocked.html
    ```
    
2. Add this html to file
    
    ```html
    <html>
    <head><title>Access Blocked</title></head>
    <body>
    <h1>Access Denied</h1>
    <p>Your request was blocked by our Web Application Firewall (WAF).</p>
    </body>
    </html>
    ```
    
3. Modify nginx site config file (/etc/nginx/sites-available/dvwa.conf
    
    ```yaml
    error_page 403 /security-blocked.html;
    
    location = /security-blocked.html {
        root /usr/share/nginx/html;
        internal;
    }
    ```
    

### Enable SSL/TLS in reverse proxy

1. Generate self-signed certificate
    
    ```bash
    sudo openssl req -x509 -nodes -days 365 \
      -newkey rsa:2048 \
      -keyout /etc/ssl/private/selfsigned.key \
      -out /etc/ssl/certs/selfsigned.crt
    ```
    
2. Modify config file like that
    
    ```yaml
    server {
        listen 443 ssl;
        server_name dvwa.com;
    
        ssl_certificate     /etc/ssl/certs/selfsigned.crt;
        ssl_certificate_key /etc/ssl/private/selfsigned.key;
    
        modsecurity on;
        modsecurity_rules_file /etc/nginx/modsec/modsecurity.conf;
    
        location /DVWA/ {
            proxy_pass http://192.168.0.108:80/DVWA/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
    
            proxy_redirect off;
        }
    
        error_page 403 /security_block.html;
    
            location = /security_block.html {
                root /usr/share/nginx/html;
                internal;
    }
    
    }
    ```
    
3. Add redirection to https
    
    ```yaml
    server {
        listen 80;
        server_name dvwa.com;
    
        return 301 https://$host$request_uri;
    }
    ```