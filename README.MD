# Presentation documentation

## Technology used

1. GitLab CE 
2. Jenkins
3. Sonarqube CE
4. Kubeadm
5. MetalLB
6. NGINX Ingress
7. Helm

## Setup gitLab CE

On devops-server node

```bash
sudo apt update
sudo apt install -y curl openssh-server ca-certificates
sudo apt install -y postfix
```
```bash
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo apt-get install gitlab-ce
gitlab-ctl start
```
user= root
pass= sudo cat /etc/gitlab/initial_root_passwor

## Setup kubeadm cluster
### Installing docker

On master node and worker node

```bash
sudo apt-get update
curl -fSsl https://get.docker.com/ | sh

cat <<EOF | sudo tee /etc/docker/daemon.json
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2"
}
EOF 

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### Install kubernetes tools (kubeadm, kubelet, kubectl)

On master node and worker node

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg
https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb
[signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg]
https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee
/etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### Inisiasi cluster

On master node

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
--control-plane-endpoint=10.8.60.211

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

On worker node

```bash
sudo kubeadm join 10.8.60.211:6443 --token l996hi.ayds99dfxzpfqupc --discovery-token-ca-cert-hash sha256:91c3eca6284d94b905ee573b83e34c378942e120b465988d0133a56ad31ce48a
```

### Setup network

On master node

```bash
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
```

### Finalisasi cluster

On master node

```bash
kubectl label node node-01 node-role.kubernetes.io/worker=worker
kubectl taint node master node-role.kubernetes.io/master:NoSchedule
```

### Setup helm

On master node

```bash
sudo apt install apt-transport-https --yes
curl https://baltocdn.com/helm/signing.asc | sudo apt-key add -
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt install -y helm
```

## Setup NFS server and storage class

### Create and setup NFS server

On NFS server node

```bash
sudo systemctl status nfs-server
sudo apt install nfs-kernel-server nfs-common portmap
sudo start nfs-server
mkdir -p /srv/nfs/mydata 
chmod -R 777 /srv/nfs/

sudo echo "/srv/nfs/mydata  *(rw,sync,no_subtree_check,no_root_squash,insecure)" >> /etc/exports
sudo exportfs -rv
showmount -e
```

On master Node

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/

helm install nfs nfs-subdir-external-provisioner/nfs-subdir-external-provisioner --set nfs.server=10.8.60.210 --set nfs.path=/srv/nfs/mydata --set storageClass.name=nfs --set storageClass.defaultClass=true -n nfs --create-namespace
```

### Setup MetalLB for Load Balancer

On master node

```bash
helm repo add metallb https://metallb.github.io/metallb
helm repo update

vim address-pool.yaml
configInline:
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 10.8.60.214-10.8.60.216

helm install metallb metallb/metallb -f address-pool.yaml -n metallb --create-namespace
```

### Setup NGINX ingress

On master node

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress ingress-nginx/ingress-nginx --set controller.service.loadBalancerIP=10.8.60.214 -n ingress --create-namespace
```

kubectl create deployment nginx --image=nginx
kubectl expose deploy nginx --port 80 --type LoadBalancer

### Deploy nginx-app

On master node

```bash
kubectl create deploy nginx-app --image=nginx
kubectl expose deploy nginx-app --type=LoadBalancer --port=80
```

### Deploy mysql & wordpress

On master node

```bash
wget https://raw.githubusercontent.com/agambewe/mysql-wp-demo/main/mysql-deployment.yaml -O mysql.yaml
wget https://raw.githubusercontent.com/agambewe/mysql-wp-demo/main/wp-deployment.yaml -O wp.yaml

kubectl create namespace demo
kubectl apply -f mysql.yaml
kubectl apply -f wp.yaml
```

## Setup Jenkins

On devops-server node

```bash
sudo apt install openjdk-11-jdk
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins

sudo vim /lib/systemd/system/jenkins.service

[Service]
Environment="JENKINS_PORT=8081"

sudo systemctl restart jenkins
```

### Setup Kubernetes agent on Jenkins

- Login to jenkins web admin > going to tab Manage Jenkins > Manage plugins > Available plugins > Search "kubernetes" on search box > click checkbox "kubernetes" > Download and install > restart jenkins 

- On master node create service account for jenkins

```bash
kubectl create namespace jenkins
kubectl get ns
kubectl create sa jenkins -n jenkins
kubectl get sa -n jenkins
kubectl create rolebinding jenkins-admin-binding --clusterrole=admin --serviceaccount=jenkins:jenkins --namespace=jenkins
kubectl get ns,sa,rolebinding -n jenkins
```

- Disable Nodes
Login to jenkins web admin > going to Manage Jenkins > Manage nodes and clouds > Built-in Node > Configure

Set "Number of executors" to 0 and save 

- Configuration secret

On master node

```bash
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: jenkins-secret
  annotations:
    kubernetes.io/service-account.name: "jenkins"
```

kubectl describe secret $(kubectl describe sa jenkins -n jenkins | grep Token | awk '{print $2}') -n jenkins

and paste to 
Manage Jenkins > Manage Credentials > Jenkins > Glocal credentials > Add Credentials

- Configuration Cloud for Kubernets Pods

kubectl cluster-info

Login to jenkins web admin > going to Manage Jenkins > Manage Nodes and Clouds > Configure Clouds > Add a new cloud (Select Kubernetes) > Kubernetes Cloud details

and fill the empty text field ith the data that we got earlier

## Setup sonarqube CE

On devops-server node

- install open jdk 11

```bash
sudo apt-get install -y openjdk-11-jdk
```

- setup postgreSQL

```bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo apt update
sudo apt install postgresql postgresql-contrib -y
sudo systemctl enable postgresql
sudo systemctl start postgresql

sudo passwd postgres
su - postgres

createuser squser
psql
ALTER USER squser WITH ENCRYPTED password '******';
CREATE DATABASE sonarqube OWNER squser;
GRANT ALL PRIVILEGES ON DATABASE sonarqube to squser;
\q
exit
```

### Install sonarqube CE

```bash
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.9.8.54436.zip -O /opt/sonarqube.zip
sudo unzip /opt/sonarqube.zip -d /opt/ 
sudo rm /opt/sonarqube.zip
sudo mv /opt/sonarqube-8.9.8.54436 /opt/sonarqube
sudo useradd sonar
sudo chown -R sonar:sonar /opt/sonarqube/
sudo vim /opt/sonarqube/conf/sonar.properties

sonar.jdbc.username=pquser
sonar.jdbc.password=******
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000

sudo vim /etc/sysctl.conf

vm.max_map_count=524288

sudo sysctl -p

sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube
```

## Sonarqube and jenkins integration

### Generate token

Login to sonarqube web admin > login > going to My Account > Security > Enter token name > Generate token

### Configure sonarqube server

Login to jenkins web admin > login > going to Manage Jenkins > Configure System > Sonarqube server > Fill data > Save

### Configure scanner server

- Login to jenkins web admin > login > going to Manage Jenkins > Manage plugins > Available plugins > Search "sonarqube" on search box > click checkbox "sonarqube" > Download and install > restart jenkins 

- Going to Manage Jenkins > Global tool configuration > Sonarqube scanner > Fill data > Save

### Configure NodeJs

- Login to jenkins web admin > login > going to Manage Jenkins > Manage plugins > Available plugins > Search "NodeJs" on search box > click checkbox "NodeJs plugin" > Download and install > restart jenkins 

- Going to Manage Jenkins > Global tool configuration > NodeJS > NodeJS installations > Fill data > Save

### Add credentials

gitlab, docker.io, cluster

For docker.io & gitlab User password
- Login to jenkins web admin > going to Manage Jenkins > Manage credentials > global > Add credentials > Fill data > Save

For cluster Secret file
kubectl config view --ra
- Login to jenkins web admin > going to Manage Jenkins > Manage credentials > global > Add credentials > Fill data > Save

### Sonarqube webhook for jenkins
- Login to sonarqube web admin > Administration > Security > switch off "Enable local webhooks validation"

- Login to sonarqube web admin > Project > klik our project > Project settings > Webhooks > Fill data

### Create nodejsapp on gitlab

- Login to gitlab > new project > import > our app url

### Config jenkins pipeline

- Login to jenkins web admin > new item > pipeline > pipeline script from SCM > Fill data

### Add new quality gate

- Login to sonarqube web admin > Quality gates > Create > Fill data