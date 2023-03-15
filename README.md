# GitLab-CI/CD Pipeline for Maven / ArgoCD

### 1. Install rke cluster

```bash
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

# setup cluster access
$ mkdir ~/.kube
$ cp kube_config_cluster.yml ~/.kube/config

# add k8s bash aliases
$ cat <<EOF >> ~/.bashrc

# k8s alias

source <(kubectl completion bash)
complete -o default -F __start_kubectl k

alias k=kubectl
alias vi=vim
alias kn='kubectl config set-context --current --namespace'
alias kc='kubectl config use-context'
alias kcg='kubectl config get-contexts'
alias di='docker images --format "table {{.Repository}}:{{.Tag}}\t{{.ID}}\t{{.Size}}\t{{.CreatedSince}}"'
alias kge="kubectl get events  --sort-by='.metadata.creationTimestamp'  -o 'go-template={{range .items}}{{.involvedObject.name}}{{\"\t\"}}{{.involvedObject.kind}}{{\"\t\"}}{{.message}}{{\"\t\"}}{{.reason}}{{\"\t\"}}{{.type}}{{\"\t\"}}{{.firstTimestamp}}{{\"\n\"}}{{end}}'"
EOF

$ source ~/.bashrc

# Install kubectl / helm
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod 755 kubectl && sudo mv kubectl /usr/local/bin
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 2. Install rancher

```bash
# Install cert-manager
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml

$ kn cert-manager
$ k rollout status deploy

# Install Rancher
$ helm repo add rancher-latest https://releases.rancher.com/server-charts/latest   
$ helm install rancher rancher-latest/rancher \
  --set hostname=rancher.vm01 \
  --set bootstrapPassword=admin \
  --set replicas=1 \
  --create-namespace -n cattle-system
  
# https://rancher.vm01
# bootstrap passwd : admin & change passwd
```

### 3. Install gitlab

```bash
# Install local-path storage class & set default
$ kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml
$ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Install gitlab
$ helm repo add gitlab https://charts.gitlab.io/
  
$ helm upgrade -i gitlab gitlab/gitlab \
  --set global.edition=ce \
  --set global.hosts.domain=herosonsa.co.kr \
  --set global.minio.enabled=false \
  --set global.ingress.configureCertmanager=false \
  --set global.ingress.class=nginx \
  --set certmanager.install=false \
  --set nginx-ingress.enabled=false \
  --set gitlab-runner.install=false \
  --set prometheus.install=false

# get root initial password
$ k get secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 -d

# login gitlab.vm01 as root / %YOUR_INITIAL_PASSWORD%

# create User with YOUR ID / PASSWD

# approve YOUR ID with root account admin menu

# Import source / deploy repository from gitlab
- https://github.com/flytux/kw-mvn
- https://github.com/flytux/kw-mvn-deploy

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

# Delete default ingress gitlab-webservice
$ k delete ingress gitlab-webservice-default -n gitlab

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


# Add selfsigned CA crt to gitlab runner via secret
# add to /etc/hosts 
%YOUR_INTERNAL_IP% gitlab.vm01

$ openssl s_client -showcerts -connect gitlab.vm01:443 -servername gitlab.vm01 < /dev/null 2>/dev/null | openssl x509 -outform PEM > gitlab.vm01.crt

$ k create secret generic gitlab-runner-tls --from-file=gitlab.vm01.crt  -n gitlab

# Install gitlab-runner with gitlab-certs using secret
$ helm fetch gitlab/gitlab-runner --untar

# add runner-cache with pvc
$ vi gitlab-runner/values.yaml

  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "ubuntu:16.04"
    [[runners.kubernetes.volumes.pvc]]
      mount_path = "/cache/maven.repository"
      name = "gitlab-runner-cache-pvc"

# create gitlab runner cache pvc
$ kubectl -n gitlab apply -f - <<"EOF"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitlab-runner-cache-pvc
  namespace: gitlab
spec:
  storageClassName: local-path
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# add in cluster dns gitlab.vm01 to coredns
$ k edit cm -n kube-system coredns
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        hosts {
          172.100.100.101 gitlab.vm01 docker.vm01 argocd.vm01
          fallthrough
        }

# get runner token from KW-MVN project
# Settings > CI/CD > Runners > Registration Token

# install gitlab-runner helm chart
$ helm upgrade -i gitlab-runner -f gitlab-runner/values.yaml gitlab-runner \
  --set gitlabUrl=https://gitlab.vm01 \
  --set runnerRegistrationToken=%YOUR-REG-TOKEN-HERE% \
  --set rbac.create=true \
  --set certsSecretName=gitlab-runner-tls
```  

### 4. install argocd & docker registry

```bash
# install argocd
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# add argocd ssl-passthrough to nginx ingress
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

# add gitlab ca-cert (self-signed)
- https://argocd.vm01/settings/certs?addTLSCert=true
- add name gitlab.vm01 & paste gitlab.vm01.crt pem file

$ cat gitlab.vm01.crt

# add argocd app 
$ kubectl -n argocd apply -f - <<"EOF"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kw-mvn
  namespace: argocd
spec:
  destination:
    namespace: deploy
    server: https://kubernetes.default.svc
  project: default
  source:
    directory:
      jsonnet: {}
      recurse: true
    path: .
    repoURL: https://gitlab.vm01/jaehoon/kw-mvn-deploy.git
    targetRevision: main
  syncPolicy:
    syncOptions:
    - CreateNamespace=true
EOF

# install docker registry
$ helm repo add twuni https://helm.twun.io
$ helm upgrade -i docker twuni/docker-registry \
  --set ingress.enabled=true \
  --set ingress.hosts[0]=docker.vm01 \
  --create-namespace -n registry
  
# set proxy-body-size of docker.vm02 ingress to 0
$ kubectl patch ingress docker-docker-registry -p '{"metadata": {"annotations":{"nginx.ingress.kubernetes.io/proxy-body-size":"0"}}}' -n registry

# configure insecure docker registry from vm01
$ sudo vi /etc/docker/daemon.json
{
   "insecure-registries": [ "docker.vm01" ]
}

$ sudo systemctl restart docker

$ curl -v docker.vm01/v2/_catalog
$ docker login docker.vm01 -u admin
```

### 5. develop build script

```bash

# Create Personal Access Token of User for deploy / Update DEPLOY_REPO_CREDENTIALS
# Unprotect main branch push from gitlab - https://gitlab.vm01/jaehoon/kw-mvn-deploy.git
# Update ARGO_USER_PASSWORD with value of kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=/cache/maven.repository"
  IMAGE_URL: "docker.vm01/kw-mvn"
  DEPLOY_REPO_URL: "https://gitlab.vm01/jaehoon/kw-mvn-deploy.git"
  DEPLOY_REPO_CREDENTIALS: "https://jaehoon:glpat-mFT6GgL_4mvDuTo8Lpmf@gitlab.vm01/jaehoon/kw-mvn-deploy.git"
  REGISTRY_USER_ID: "admin"
  REGISTRY_USER_PASSWORD: "1"
  ARGO_URL: "argocd.vm01"
  ARGO_USER_ID: "admin"
  ARGO_USER_PASSWORD: "nUBniE6vQv1Sy1JH"
  ARGO_APP_NAME: "kw-mvn"

stages:
  - maven-jib-build
  - update-yaml
  - sync-argo-app

maven-jib-build: 
  image: gcr.io/cloud-builders/mvn@sha256:57523fc43394d6d9d2414ee8d1c85ed7a13460cbb268c3cd16d28cfb3859e641
  stage: maven-jib-build
  script:
    - COMMIT_TIME="$(date -d "$CI_COMMIT_TIMESTAMP" +"%Y%m%d-%H%M%S")"
    - "mvn -B \
        -DsendCredentialsOverHttp=true \
        -Djib.allowInsecureRegistries=true \
        -Djib.to.image=$IMAGE_URL:$COMMIT_TIME-$CI_JOB_ID \
        -Djib.to.auth.username=$REGISTRY_USER_ID \
        -Djib.to.auth.password=$REGISTRY_USER_PASSWORD     \
        compile \
        com.google.cloud.tools:jib-maven-plugin:build"
    - echo "IMAGE_FULL_NAME=$IMAGE_URL:$COMMIT_TIME-$CI_JOB_ID" >> build.env
    - cat build.env
  artifacts:
    reports:
      dotenv: build.env

update-yaml:
  image: alpine/git:v2.26.2
  stage: update-yaml
  script:
    - mkdir deploy && cd deploy
    - git init

    - echo $DEPLOY_REPO_CREDENTIALS > ~/.git-credentials 
    - cat ~/.git-credentials
    - git config credential.helper store
    
    - git remote add origin $DEPLOY_REPO_URL
    - git remote -v

    - git -c http.sslVerify=false fetch --depth 1 origin $CI_COMMIT_BRANCH
    - git -c http.sslVerify=false checkout $CI_COMMIT_BRANCH
    - ls -al

    - echo "updating image to $IMAGE_FULL_NAME"
    - sed -i "s|docker.vm01/kw-mvn:.*$|$IMAGE_FULL_NAME|" deploy.yml
    - cat deploy.yml | grep image
    
    - git config --global user.email "tekton@tekton.dev"
    - git config --global user.name "Tekton Pipeline"
    - git add .
    - git commit --allow-empty -m "[tekton] updating image to $IMAGE_FULL_NAME"
    - git -c http.sslVerify=false push origin $CI_COMMIT_BRANCH

sync-argocd:
  image: quay.io/argoproj/argocd:v2.4.8
  stage: sync-argo-app
  script:
    - argocd login $ARGO_URL --username $ARGO_USER_ID --password $ARGO_USER_PASSWORD --insecure

    - argocd app sync $ARGO_APP_NAME --insecure
    - argocd app wait $ARGO_APP_NAME --sync --health --operation --insecure
```

### 6. run pipeline

!!!

