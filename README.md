# AWX on Single Node Minikube and RockyLinux
- Persistent local database
- SSL/TLS Certificate
- Persistent projects folder

## Environment
- RockyLinux 8.7
- PostgreSQL 13
- Awx Operator 1.1.0

## Requirements
- 4vCPUs
- 8144 MiB RAM
- 25GB HDD
- SSL/TLS Certificate (key.pem and cert.pem)

## Deployment Instruction

### 1) Install and Configure PostgreSQL
```
# yum module enable postgresql:13
# yum module list postgresql
# dnf module install postgresql
# systemctl enable postgresql.service
# /usr/bin/postgresql-setup --initdb
# systemctl start postgresql.service
```
### 2) Create awx database
```
# su - postgres
$ psql -l
$ createdb awx
$ psql -l
```
### 3) Create awx user and give permissions
```
$ psql
# create user awx with encrypted password 'password';
# grant all privileges on database awx to awx;
# \q
```
### 4) Edit this line in file /var/lib/pgsql/data/pg_hba.conf
```
From :
host   all   all   127.0.0.1/32   ident

To:
host   all   all   0.0.0.0/0   md5
```
### 5) Edit this line in file /var/lib/pgsql/data/postgresql.conf
```
listen_address = '*'
```
### 6) Restart PostgreSQL
```
# systemctl restart postgresql.service
```
### 7) Configure firewalld with rich-rules
```
# firewall-cmd --zone=public --add-masquerade --permanent
# firewall-cmd --add-rich-rule='rule family="ipv4" source address="IP" port port="80" protocol="tcp" accept' --permanent
# firewall-cmd --add-rich-rule='rule family="ipv4" source address="IP" port port="443" protocol="tcp" accept' --permanent
# firewall-cmd --add-rich-rule='rule family="ipv4" source address="IP" port port="5432" protocol="tcp" accept' --permanent
# firewall-cmd --reload
```
### 8) Install docker
```
# dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
# dnf install docker-ce --nobest -y
# systemctl enable docker --now
```
### 9) Install minikube
```
# dnf install conntrack -y
# wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
# chmod +x minikube-linux-amd64
# sudo mv minikube-linux-amd64 /usr/local/bin/minikube
# minikube version
```
### 10) Install kubectl
```
# curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
# chmod +x kubectl
# sudo mv kubectl  /usr/local/bin/
# kubectl version --client -o json
```
### 11) Create awx user
```
# useradd awx
# usermod -a -G wheel awx
# usermod -a -G docker awx
# sudo su - awx
```
### 12) Start the cluster, get minikube ip and add it to /etc/hosts
```
$ minikube start --cpus=4 --memory=6g --addons=ingress
$ minikube kubectl -- get nodes
$ minikube kubectl -- get pods -A
$ minikube ip

/etc/hosts
192.168.49.2   minikube
```
### 13) Install kustomize and git
```
$ curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh"  | bash
$ chmod +x kustomize
# mv kustomize /usr/local/bin
# dnf install git -y
```
### 14) Create kustomization.yaml
```
$ vim kustomization.yaml
---BEGIN---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=1.1.0

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 1.1.0

# Specify a custom namespace in which to install AWX
namespace: awx
---END---
```
### 15) Build, Check and Change context
```
$ kustomize build . | kubectl apply -f -
$ kubectl get pods -n awx
$ kubectl config set-context --current --namespace=awx
```
### 16) Create awx.yaml, choosing IP, Port and Password for the conection with the postgres database and Hostname (same as in your ssl/tls cert) for your awx host
```
$ vim awx.yaml
---BEGIN---
---
apiVersion: v1
kind: Secret
metadata:
  name: awx-postgres-configuration
  namespace: awx
stringData:
  host: "IP"
  port: "Port"
  database: awx
  username: awx
  password: "Password"
  sslmode: prefer
  type: unmanaged
type: Opaque

---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  ingress_type: ingress
  hostname: "Hostname"
  projects_persistence: true
  projects_storage_class: rook-ceph
  service_type: nodeport
  nodeport_port: 30080
  postgres_configuration_secret: awx-postgres-configuration
---END---
```
### 17) Edit kustomization.yaml, now including the new resourse awx.yaml
```
$ vim kustomization.yaml
---BEGIN---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=1.1.0
  - awx.yaml

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: 1.1.0

# Specify a custom namespace in which to install AWX
namespace: awx
---END---
```
### 18) Build, Follow the logs till the end and Check
```
$ kustomize build . | kubectl apply -f -
$ kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
$ kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator"
```
### 19) See the URL
```
$ minikube service awx-service --url -n awx
```
### 20) Create a TLS kubernetes secret
```
# kubectl create secret tls awx-ingress-tls --key="key.pem" --cert="certificate.pem"
```
### 21) Edit the file awx.yaml, adding
```
spec:
   ingress_tls_secret: awx-ingress-tls
```
### 22) Build again
```
$ kustomize build . | kubectl apply -f -
```
### 23) Create the file ingress.yaml
```
$ vi ingress.yaml
---BEGIN---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: awx-ingress
  namespace: awx
  annotations:
spec:
  tls:
  - hosts:
      - "minikube"
    secretName: awx-ingress-tls
  rules:
  - host: "minikube"
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: awx-service
            port:
              number: 80
---END---
```
### 24) Create Ingress
```
$ kubectl create -f ingress.yaml
```
### 25) Get web admin password
```
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode
```
### 26) Create port forward from your host to the minikube ip.
```
# firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=80:toaddr=192.168.49.2 --permanent
# firewall-cmd --zone=public --add-forward-port=port=443:proto=tcp:toport=443:toaddr=192.168.49.2 --permanent
# firewall-cmd --reload
```
### 27) Test web access
```
https://Hostname
```
### 28) Create a folder inside the projects directory:
```
$ kubectl exec -it -n awx awx_pod -c awx-task — /bin/bash
$ cd /var/lib/awx/projects
$ mkdir folder_name
```
