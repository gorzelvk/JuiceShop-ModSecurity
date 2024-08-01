JuiceShop application with ModSecurity installed on nginx reverse-proxy

This document outlines the steps to set up a Kubernetes cluster with the OWASP Juice Shop application and configure ModSecurity with Core Rule Set (CRS).

## Steps Overview

1. **Launch Kubernetes cluster**
2. **Deploy OWASP Juice Shop application**
3. **Configure Nginx as reverse-proxy**
4. **Install ModSecurity on Nginx**
5. **Add CoreRuleSet to ModSecurity**
6. **Implement additional rules**



### 1. Launch local Kubernetes cluster
##### Run the following command to download the Minikube binary

	curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
	sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

##### Start your cluster:

	minikube start

### 2. Deploy OWASP Juice Shop application

##### Install JuiceShop app on kubernetes with helm (helm >= 3.7 required)
	
	helm install multi-juicer oci://ghcr.io/juice-shop/multi-juicer/helm/multi-juicer
	
##### Verify deployment
	
	kubectl get pods

##### Expose your application with Kubernetes service

	kubectl port-forward svc/juice-balancer 3000:3000

### 3. Configure Nginx as reverse-proxy

##### Install Nginx	

	sudo apt update
	sudo apt install nginx

##### Allow access to Nginx through firewall

	sudo ufw allow 'Nginx HTTP'

##### Verify that Nginx is running

	systemctl status nginx

##### Configure server block

	sudo nano /etc/nginx/sites-available/{{YOUR-DOMAIN-NAME}}
######	example: sudo nano /etc/nginx/sites-available/localhost

######	'/etc/nginx/sites-available/localhost'
	```
	server {
        listen 80;              # Nginx will listen on port 80 (HTTP)
        listen [::]:80;         # IPv6 port 80

        server_name localhost;

        location / {
                proxy_pass http://127.0.0.1:3000; # Enable reverse-proxy functionality
        	}
	}
	```

##### Verify if the configuration is working

###### Open browser and navigate to localhost - from there you should be directed to your application.
###### (Note: open your browser in incognito mode to avoid problems)

##### Configure header forwarding settings

######	'/etc/nginx/proxy_params'
	```
	proxy_set_header Host $http_host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto $scheme;
	```

##### Create softlink to our configuration from sites-enabled directory that Nginx reads at startup

	sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/

##### Restart Nginx to apply changes

	sudo systemctl restart nginx

### 4. Install ModSecurity on Nginx

##### Install dependencies required for ModSecurity build and compilation process

	sudo apt-get install bison build-essential ca-certificates curl dh-autoreconf doxygen \
	flex gawk git iputils-ping libcurl4-gnutls-dev libexpat1-dev libgeoip-dev liblmdb-dev \
  	libpcre3-dev libpcre++-dev libssl-dev libtool libxml2 libxml2-dev libyajl-dev locales \
  	lua5.3-dev pkg-config wget zlib1g-dev zlibc libxslt libgd-dev

##### Clone ModSecurity repository into /opt dir

	cd /opt && sudo git clone https://github.com/SpiderLabs/ModSecurity

##### Initialize and update submodule
	
	cd ModSecurity	
	sudo git submodule init
	sudo git submodule update
	
##### Run build.sh script

	sudo ./build.sh

##### Run configure file to get dependencies required for build process

	sudo ./configure

##### Build ModSecurity

	sudo make
	
##### Install ModSecurity
	
	sudo make install

##### Download ModSecurity-Nginx Connector

######	ModSecurity-Nginx Connector is a module for Nginx that integrates ModSecurity, a Web Application Firewall (WAF),
######	with the Nginx web server. It acts as a bridge, allowing ModSecurity to analyze HTTP requests and apply security rules
######	directly within the Nginx environment.

##### Clone ModSecurity-Nginx connector into /opt dir

	cd /opt && sudo git clone --depth 1 https://github.com/SpiderLabs/ModSecurity-nginx.git

##### Download exact version of Nginx that is running on your system into /opt dir and extract the tarball

	cd /opt && sudo wget http://nginx.org/download/nginx-{{NGINX-VERSION}}.tar.gz
	sudo tar -xvzmf nginx-{{NGINX-VERSION}}.tar.gz

##### Display configure arguments used for Nginx

	nginx -V

######	Example output for Nginx:1.18.0

	```
	nginx version: nginx/1.18.0 (Ubuntu)
	built with OpenSSL 3.0.2 15 Mar 2022
	TLS SNI support enabled
	configure arguments: --with-cc-opt='-g -O2 -ffile-prefix-map=/build/nginx-zctdR4/nginx-1.18.0=. -flto=auto -ffat-lto-objects -flto=auto \
	-ffat-lto-objects -fstack-protector-strong -Wformat -Werror=format-security -fPIC -Wdate-time -D_FORTIFY_SOURCE=2' --with-ld-opt='-Wl, \
	-Bsymbolic-functions -flto=auto -ffat-lto-objects -flto=auto -Wl,-z,relro -Wl,-z,now -fPIC' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf \
	--http-log-path=/var/log/nginx/access.log --error-log-path=/var/log/nginx/error.log --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid \
	--modules-path=/usr/lib/nginx/modules --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
	--http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-compat \
  	--with-debug --with-pcre-jit --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module \
	--with-http_v2_module --with-http_dav_module --with-http_slice_module --with-threads --add-dynamic-module=/build/nginx-zctdR4/nginx-1.18.0/debian/modules/http-geoip2 \
	--with-http_addition_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_sub_module
	```

##### Compile ModSecurity module with configure arguments
		
	sudo ./configure --add-dynamic-module=../ModSecurity-nginx <Configure Arguments>

##### Build modules

	sudo make modules

##### Copy compiled ModSecurity module into Nginx configuration directory

	sudo mkdir /etc/nginx/modules
	sudo cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules

##### Load ModSecurity Module in Nginx
###### Copy the following into /etc/nginx/nginx.conf:

	load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;

### 5. Add CoreRuleSet to ModSecurity

###### Delete default rule set

	sudo rm -rf /usr/share/modsecurity-crs

###### Clone CoreRuleSet repository into /usr/share/modsecurity-crs

	sudo git clone https://github.com/coreruleset/coreruleset /usr/local/modsecurity-crs

###### Remove the .example extensions of crs-setup.conf and exclusion rule file

	sudo mv /usr/local/modsecurity-crs/crs-setup.conf.example /usr/local/modsecurity-crs/crs-setup.conf
	sudo mv /usr/local/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example /usr/local/modsecurity-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf

###### Create ModSecurity directory in /etc/nginx/ directory

	sudo mkdir -p /etc/nginx/modsec

###### Copy unicode mapping file and ModSecurity configuration file into /modsec directory

	sudo cp /opt/ModSecurity/unicode.mapping /etc/nginx/modsec
	sudo cp /opt/ModSecurity/modsecurity.conf-recommended /etc/nginx/modsec/modsecurity.conf

###### Change default SecRuleEngine value

######  '/etc/nginx/modsec/modsecurity.conf'
	```

	...
        SecRuleEngine On
	...

	```	

###### Create main configuration file under /etc/nginx/modsec directory

	sudo touch /etc/nginx/modsec/main.conf

###### Specify rules and ModSecurity configuration file for Nginx

	Include /etc/nginx/modsec/modsecurity.conf
	Include /usr/local/modsecurity-crs/crs-setup.conf
	Include /usr/local/modsecurity-crs/rules/*.conf

###### Insert following lines into http block in /etc/nginx/nginx.conf configuration file

###### 	'/etc/nginx/nginx.conf'	

###### 	Load ModSecurity configuration files into Nginx

	modsecurity on; 
	modsecurity_rules_file /etc/nginx/modsec/main.conf;
	include /etc/nginx/conf.d/*.conf; 
	include /etc/nginx/sites-enabled/*;

######	Specify log files path

	access_log /var/log/nginx/access.log; 
	error_log /var/log/nginx/error.log;

### 6. Implement additional rules
