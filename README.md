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

### 4. Install ModSecurity on Nginx

### 5. Add CoreRuleSet to ModSecurity

### 6. Implement additional rules
