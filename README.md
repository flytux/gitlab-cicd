# gitlab-cicd


### 1. Install rke cluster

~~~
# Install Docker
$ curl -fsSL https://get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh

# Install user key
$ ssh-keygen
$ ssh-copy-id k8sadm@vm01 # User@Host

# RKE binary download
$ wget https://github.com/rancher/rke/releases/download/v1.3.15/rke_linux-amd64
$ chmod 755 rke_linux-amd64 && sudo mv rke_linux-amd64 /usr/local/bin/rke

# RKE cluster install
$ rke config
# host address : vm01, user : k8sadm, etcd/control/worker : y
$ rke up
   
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
  
$ helm upgrade -i gitlab gitlab/gitlab \
  --set global.hosts.domain=vm01 \
  --set certmanager.install=false \
  --set global.ingress.configureCertmanager=false \
  --set global.ingress.class=nginx \
  --set nginx-ingress.enabled=false \
  --set gitlab-runner.install=false \
  --set prometheus.install=false \
  --create-namespace -n gitlab

# Gitlab Runner doesn't support autogenerated certs from gitlab for now
# Create CA certs, Ingress TLS from cert-manager & pass it to gitlab-runner certs-secret

# Create CA certs for CA Issuer
$ openssl genrsa -out ca.key 2048
$ openssl req -new -x509 -days 3650 -key ca.key -subj "/C=KR/ST=SE/L=SE/O=Kubeworks/CN=KW Root CA" -out ca.crt
$ kubectl create secret tls gitlab-ca --key ca.key --cert ca.crt -n gitlab

# Create CA Issuer
$ kubectl -n gitlab apply -f - <<"EOF"
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: gitlab-ca-issuer
  namespace: gitlab
spec:
  ca:
    secretName: gitlab-ca
EOF

# Create Ingress 
$ kubectl -n gitlab apply -f - <<"EOF"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/issuer: gitlab-ca-issuer
  name: gitlab-web-ingress
  namespace: gitlab
spec:
  ingressClassName: nginx
  rules:
  - host: gitlab.vm01
    http:
      paths:
      - backend:
          service:
            name: gitlab-webservice-default
            port:
              number: 8181
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - gitlab.vm01
    secretName: gitlab-web-tls
EOF

$ openssl s_client -showcerts -connect gitlab.vm01:443 -servername gitlab.vm01 < /dev/null 2>/dev/null | openssl x509 -outform PEM > gitlab.vm01.crt

$ k create secret generic gitlab-runner-tls --from-file=gitlab.vm01.crt  -n gitlab

# Install gitlab-runner with gitlab-certs using secret

$ helm upgrade -i gitlab-runner gitlab/gitlab-runner \
  --set gitlabUrl=https://gitlab.vm01 \
  --set runnerRegistrationToken=%YOUR-REG-TOKEN-HERE% \
  --set rbac.create=true \
  --set certsSecretName=gitlab-runner-tls
 
~~~

### 4. install argocd

~~~
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

$ k edit ds -n ingress-nginx nginx-ingress-controller

# add "--enable-ssl-passthrough" end of containers.args like below
  - --watch-ingress-without-class=true
  - --enable-ssl-passthrough

# add ingress for argocd

$ kubectl -n argocd apply -f - <<"EOF"  
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
spec:
  rules:
  - host: argocd.vm01
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              name: https
EOF

# get argocd initial password

$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
~~~

### 5. develop build script

~~~
$ helm upgrade -i docker twuni/docker-registry \
  --set ingress.enabled=true \
  --set ingress.hosts[0]=docker.vm01 \
  --create-namespace -n registry
  
$ vi /etc/docker/daemon.json
{
   "insecure-registries": [ "docker.vm01", "172.100.100.101:30005" ]
}

$ sudo docker restart 
~~~

**gitlab-runner pipeline script - Work In Progress**

~~~
variables:
  MAVEN_OPTS: "-Dmaven.repo.local=/cache/maven.repository"

stages:
  - build-id          # List of stages for jobs, and their order of execution
  - maven-jib-build
  - update-yaml

get-build-id:
  image: docker.io/library/bash:5.0.18@sha256:8ef3f8518f47caf1ddcbdf49e983a9a119f9faeb41c2468dd20ff39cd242d69d #tag: 5.0.18
  stage: build-id
  script:
    - ts=`date "+%y%m%d-%H%M%S"`
    - 'echo \"Current Timestamp: ${ts}\"'
    - base='dev'
    - id=`echo $RANDOM | md5sum | head -c 8`
    - buildId=${base}-${ts}-${id}
    - echo ${buildId}
    - echo "BUILD_ID=$buildId" >> build.env
    - cat build.env
  artifacts:
    reports:
      dotenv: build.env

maven-jib-build:       # This job runs in the build stage, which runs first.
  image: gcr.io/cloud-builders/mvn@sha256:57523fc43394d6d9d2414ee8d1c85ed7a13460cbb268c3cd16d28cfb3859e641
  stage: maven-jib-build
  script:
    - "mvn -B \
        -DsendCredentialsOverHttp=true \
        -Djib.allowInsecureRegistries=true \
        -Djib.to.image=docker.vm01/kw-mvn:$BUILD_ID \
        -Djib.to.auth.username=admin \
        -Djib.to.auth.password=1     \
        compile \
        com.google.cloud.tools:jib-maven-plugin:build"
    - echo "IMAGE_URL=docker.vm01/kw-mvn:$BUILD_ID" >> build.env
    - cat build.env
  artifacts:
    reports:
      dotenv: build.env

update-yaml:
  image: alpine/git:v2.26.2
  stage: update-yaml
  script:
    - rm -rf ./*
    - rm -rf ./.[!.]*
    - rm -rf ./..?*
    - git init
    - git remote add origin https://gitlab.vm01/jaehoon/kw-mvn-deploy.git
    - git fetch --depth 1 origin main
    - git checkout -b main
~~~

### 6. run pipeline
