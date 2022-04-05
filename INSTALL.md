## Installation guide
1. [Install Linux Update](#Install-Linux-Update)
2. [Install Minikube Cluster](#Install-Minikube-Cluster)
3. [Install Ansible AWX](#Install-Ansible-AWX)
4. [Install Nginx](#Install-Nginx)
5. [Install Proxmoxer](#Install-Proxmoxer)
6. [FAQs](#faqs)
7. [Sources](#sources)

### Install Linux Update
***
```
sudo apt update -y
sudo apt upgrade -y
```

### Install Minikube Cluster
***
**Install Minikube dependencies**
```
sudo apt install -y curl wget apt-transport-https
```
**Install Minikube**
```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube

sudo cp minikube /usr/local/bin && rm minikube
```
**Check Minikube Version**
```
localadmin@ansible:~$ minikube version
minikube version: v1.24.0
commit: 76b94fb3c4e8ac5062daf70d60cf03ddcc0a741b
```

**Install Kubectl utility**
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```
**Check Kubectl utility**
```
localadmin@ansible:~$ kubectl version -o yaml
clientVersion:
  buildDate: "2021-12-16T11:41:01Z"
  compiler: gc
  gitCommit: 86ec240af8cbd1b60bcc4c03c20da9b98005b92e
  gitTreeState: clean
  gitVersion: v1.23.1
  goVersion: go1.17.5
  major: "1"
  minor: "23"
  platform: linux/amd64

The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
**Install Docker Engine**

```
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt-get install docker-ce docker-ce-cli containerd.io

sudo usermod -aG docker $USER && newgrp docker
```
**Set namespace environment variable**
```
export NAMESPACE=mtma-awx
echo export NAMESPACE=mtma-awx >> .profile
```
**Create systemd unit service for minikube**
```
sudo nano /etc/systemd/system/minikube.service
```
```
[Unit]
Description=minikube
After=sshd.service

[Service]
Type=oneshot
RemainAfterExit=yes
User=user
WorkingDirectory=/home/user
ExecStart=/usr/local/bin/minikube start --cpus=4 --memory=6g --addons=ingress --driver=docker --namespace=mtma-awx
ExecStop=/usr/local/bin/minikube stop

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl enable minikube.service
```
```
sudo systemctl start minikube.service
```
**Check minikube Status**
```
localadmin@ansible:~$ minikube status
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```
```
localadmin@ansible:~$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

```
localadmin@ansible:~$ kubectl get nodes
NAME       STATUS   ROLES                  AGE    VERSION
minikube   Ready    control-plane,master   100s   v1.22.3
```
### Install Ansible AWX
***
**Set Alias for kubectl**
```
alias kubectl="minikube kubectl --"
```
**Install build-essential**
```
sudo apt-get install build-essential
```
**Download Ansible AWX**
```
git clone https://github.com/ansible/awx-operator.git
cd awx-operator/
git checkout 0.15.0
```
**Deploy Ansible AWX**
```
cd awx-operator/
make deploy
```
**Check AWX Deployment**
```
kubectl get pods -n $NAMESPACE
```
**Set default namespace**
```
kubectl config set-context --current --namespace=$NAMESPACE
```
**Create deployment file**
```
nano mtma-awx.yml
```
```
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: mtma-awx
spec:
  service_type: nodeport
  nodeport_port: 8080
  ingress_type: none
  hostname: awx.fst91.ein.dev
  admin_user: admin
  admin_email: admin@ein.dev
```
**Apply deployment file**
```
kubectl apply -f mtma-awx.yml
```
**Check deployment's logs**
```
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
```
**Check AWX pods**
```
kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator"
```
**Check AWX service**
```
kubectl get svc -l "app.kubernetes.io/managed-by=awx-operator"
```
**Get AWX Url**
```
user@master:/etc/systemd/system$ minikube service mtma-awx-service --url -n $NAMESPACE
http://192.168.49.2:31105
```
### Install Nginx
***
**Install certbot to create and renew SSL X.509 Certificat**
```
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```
**Install Nginx via APT**
```
sudo apt install nginx
```
**Create SSL certificat**
```
sudo certbot --nginx -d awx.mtma.ein.dev -d www.awx.mtma.ein.dev
```
**Reconfigure Nginx HTTPS listner**
```
sudo nano /etc/nginx/sites-available/default
```
```
 location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                #try_files $uri $uri/ =404;
                proxy_pass  http://192.168.49.2:31105/;
        }
        location /websocket {
                proxy_pass  http://192.168.49.2:31105/websocket;
                proxy_http_version 1.1;
        proxy_set_header   Host               $host:$server_port;
        proxy_set_header   X-Real-IP          $remote_addr;
        proxy_set_header   X-Forwarded-For    $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto  $scheme;
        proxy_set_header   Upgrade            $http_upgrade;
        proxy_set_header   Connection         "upgrade";
        }
```

**Reload Nginx service**
```
systemctl reload nginx.service
```
### Install Proxmoxer
***
**Install python3**
```
sudo apt install python-is-python3
```
**Install pip**
```
sudo apt install python3-pip
```
**Install Proxmoxer**
```
pip install proxmoxer
pip install requests
pip install paramiko
pip install openssh_wrapper
```
**Install Proxmox Ansible module**
```
ansible-galaxy collection install community.general
```
### FAQs
***
**How can I get the password of the admin user after install?**
```
kubectl get secret mtma-awx-admin-password -o jsonpath="{.data.password}" | base64 --decode
```
**How can I get the database user?**
```
kubectl get secret -n mtma-awx mtma-awx-postgres-configuration -o jsonpath="{.data.username}" | base64 --decode
```
**How can I get the database password?**
```
kubectl get secret -n mtma-awx mtma-awx-postgres-configuration -o jsonpath="{.data.password}" | base64 --decode
```

### Sources
***
A list of sources used within the project:
* [Install minikube](https://kubernetes.io/de/docs/tasks/tools/install-minikube/#linux)
* [Install docker](https://docs.docker.com/engine/install/ubuntu/)
* [Install Ansible AWX](https://github.com/ansible/awx-operator)
* [Install Nginx](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)
* [Install certbot](https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal)