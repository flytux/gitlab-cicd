# gitlab-cicd


### 1. Install rke cluster

~~~
# Install Docker
$ sudo zypper in -y docker

# Install user key
$ ssh-keygen
$ ssh-copy-id k8sadm@vm01 # User@Host

# RKE binary download
$ wget https://github.com/rancher/rke/releases/download/v1.3.15/rke_linux-amd64
$ chmod 755 rke_linux-amd64 && sudo mv rke_linux-amd64 /usr/local/bin/rke

# RKE cluster install
$ rke config
   
# Install kubectl / helm   
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod 755 kubectl && sudo mv kubectl /usr/local/bin
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
~~~

### 2. Install rancher

~~~

# Install cert-manager
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml

# Install Rancher
$ helm repo add rancher-latest https://releases.rancher.com/server-charts/latest   
$ helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.vm01 \
  --set bootstrapPassword=admin \
  --create-namespace
~~~

### 3. Install gitlab

~~~
# Install local-path storage class & set default
$ kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml
$ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Install gitlab
$ helm repo add gitlab https://charts.gitlab.io/
$ helm install gitlab gitlab/gitlab \
  --set hosts.domain=vm01 \
  --set certmanager.install=false \
  --set global.ingress.configureCertmanager=false \
  --set nginx-ingress.enabled=false \
  --set gitlab-runner.install=false
~~~

### 4. install argocd

### 5. install gitlab kubernetes integration

### 6. develop build script

### 7. run pipeline
