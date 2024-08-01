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

	* Run the following command to download the Minikube binary

	curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
	sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

	* Start your cluster:

	minikube start

### 2. Deploy OWASP Juice Shop application

	* Install JuiceShop app on kubernetes with helm (helm >= 3.7 required)
	
	helm install multi-juicer oci://ghcr.io/juice-shop/multi-juicer/helm/multi-juicer
	
	* Verify deployment
	
	kubectl get pods

	* Expose your application with Kubernetes service

	kubectl port-forward svc/juice-balancer 3000:3000

### 3. Configure Nginx as reverse-proxy

	* Install Nginx	

	sudo apt update
	sudo apt install nginx

	* Allow access to Nginx through firewall

	sudo ufw allow 'Nginx HTTP'

	* Verify that Nginx is running

	systemctl status nginx

	* Configure server block

	sudo nano /etc/nginx/sites-available/{{YOUR-DOMAIN-NAME}}
	example: sudo nano /etc/nginx/sites-available/localhost

	'/etc/nginx/sites-available/localhost'
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

	* Verify if the configuration is working

	Open browser and navigate to localhost - from there you should be directed to your application.
	(Note: open your browser in incognito mode to avoid problems)

	* Configure header forwarding settings

	'/etc/nginx/proxy_params'
	```
	proxy_set_header Host $http_host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_set_header X-Forwarded-Proto $scheme;
	```

	* Create softlink to our configuration from sites-enabled directory that Nginx reads at startup

	sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/

	* Restart Nginx to apply changes

	sudo systemctl restart nginx

### 4. Install ModSecurity on Nginx

### 5. Add CoreRuleSet to ModSecurity

### 6. Implement additional rules
