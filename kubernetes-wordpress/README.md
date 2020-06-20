# Wordpress--Mysql Kubernetes Cluster

This project aims to build an wordpress kubernetes cluster with mysql database on Ubuntu using flannel network solution.
We are using two nodes only, master and worker.

# Install Kuernetes cluster 
## Pre-Config
```bash
master# hostnamectl set-hostname master.project.com
worker# hostnamectl set-hostname worker.project.com
```
## Install Docker
```bash
 sudo apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common

 curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

  sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

 sudo apt-get update

 sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu

 sudo apt-mark hold docker-ce

 service docker start

 sudo docker version

```
## Install Kubernetes
```bash

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update

sudo apt-get install -y kubelet=1.12.7-00 kubeadm=1.12.7-00 kubectl=1.12.7-00

sudo apt-mark hold kubelet kubeadm kubectl

service kubelet start 

kubeadm version
```
## Prepare UFW Firewall 
```bash
ufw enable && service ufw start

master# ufw allow 22,80,443,8080,3306,6443,2379:2380,10250:10252,10255/tcp

worker# ufw allow 22,80,443,8080,3306,10250:10251,10255,30000:32767/tcp

ufw status verbose
```

## Update Iptables Settings
```bash
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf

sudo sysctl -p

sudo sysctl --system

```

## Disable SWAP
```bash
sudo sed -i '/swap/d' /etc/fstab

sudo swapoff -a
```

## Create Kubernetes User
```bash
master# sudo useradd k8s -md /home/k8s -s /bin/bash

master# sudo passwd k8s

master# echo "k8s ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

master# sudo su - k8s
```
## Kubernetes Cluster
```bash
master# sudo kubeadm init --pod-network-cidr=10.244.0.0/16

master# mkdir -p $HOME/.kube

master# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

master# sudo chown $(id -u):$(id -g) $HOME/.kube/config

worker#  kubeadm join $some_ip:6443 --token $some_token --discovery-token-ca-cert-hash $some_hash
```
## Insatll Flannel Network
```bash
master# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.12.0/Documentation/kube-flannel.yml

master# kubectl get nodes

master# kubectl describe nodes
```

# Start with Secert 
create secret to define database name, database username and password.

Values of secert are add in base64 format, These are how we got the base64 format.
```bash 
echo -n '123456' | base64
MTIzNDU2

echo -n 'wordpress_user' | base64
d29yZHByZXNzX3VzZXI=

echo -n 'wordpress_db' | base64
d29yZHByZXNzX2Ri

```
### mysql-secret.yaml

```yaml
#======================================= Define Secret ======================================
apiVersion: v1
kind: Secret
metadata:
  name: mysql-pass
type: Opaque
data:
  username: d29yZHByZXNzX3VzZXI=
  password: MTIzNDU2
  dbname: d29yZHByZXNzX2Ri

```
# MySQL
Service wordpress of type LoadBalancer is listening on port 3306.
Deployment wordpress making sure that 2 replicas of wordpress pods are running.
Secret parameters are passed in env variables.
Storage of type of hostPath.
### mysql-deployment.yaml
```yaml
#=========================================== Define Service ==================================
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
#======================================= Mysql Deployment ===================================
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: a7medelgbaly/iti-graduation-project:mysql
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-volume
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-volume
        hostPath:
          path: /home/k8s/ITI-Graduation-Project/k8s-manifest/mysql-volume/
          type: DirectoryOrCreate

```

# Wordpress
Service wordpress of type LoadBalancer is listening on port 31204 and forward data to port 80/tcp deployment for http wordpress managment.
Deployment wordpress making sure that 2 replicas of wordpress pods are running.
Secret parameters are passed in env variables.
Storage of type of hostPath.

### wordpress-deployment.yaml
```yaml
#=========================================== Define Service ==================================
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
  - nodePort: 31204
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
#======================================= Mysql Deployment ===================================
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:php7.2
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: username
        - name: WORDPRESS_DB_NAME
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: dbname
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-volume
          mountPath: /var/www/html/
      volumes:
      - name: wordpress-volume
        hostPath:
          path: /home/k8s/ITI-Graduation-Project/k8s-manifest/wordpress-volume/
          type: DirectoryOrCreate

```
