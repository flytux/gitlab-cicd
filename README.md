# GitLab-CI/CD Pipeline for Maven / ArgoCD

- 4 Core, 16Gi, 100GB VM 3기를 준비합니다.
- 2기는 DevOps 클러스터, 1기는 서비스 클러스터로 2개의 클러스터를 구성합니다.
- DevOps 클러스터에 Rancher, Gitlab, Gitlab Runner, ArgoCD 등 툴체인을 구성합니다.
- 빌드된 서비스는 서비스 클러스터로 배포됩니다.

### 1. Install rke2 devops cluster

- VM1에 rke2 마스터노드를 설치하고, 클러스터 추가용 token을 확인합니다.
- VM2에 rke2 워커노드를 설치하고 마스터노드에 연결합니다.

```bash
# Create RKE2 cluster Master Node @ VM1

# Login VM1

$ sudo -i

$ curl -sfL https://get.rke2.io | sh -

$ systemctl enable rke2-server --now &

$ systemctl status -l rke2-server

$ journalctl -fa

# Check server IP & token

$ ip a | grep inet

# Check master node token to join the cluster
$ cat /var/lib/rancher/rke2/server/token
K107a5ebf3e93c0ce43b8c83be33eebd556470ba242dd471e393bf51415e63d4590::server:4e597066002aae1dc9770aa81a38104a

# Create RKE2 cluster Worker Node @ VM2

# Login VM2

$ sudo -i

$ curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -

$ mkdir -p /etc/rancher/rke2/

# Ediit master node IP and master node token to join
$ cat << EOF >> /etc/rancher/rke2/config.yaml
server: https://10.128.15.213:9345 # VM1 private IP 
token: K107a5ebf3e93c0ce43b8c83be33eebd556470ba242dd471e393bf51415e63d4590::server:4e597066002aae1dc9770aa81a38104a
EOF

$ systemctl enable rke2-agent.service --now &

$ journalctl -fa
```
---

### 2. Setup cluster access

- VM1에 클러스터 접속을 위한 kubeconfig 환경을 설정합니다.

```bash

# Login VM1

$ mkdir ~/.kube
$ sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
$ sudo chown k8sadm ~/.kube/config 

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

# Install kubectl / helm
$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ chmod 755 kubectl && sudo mv kubectl /usr/local/bin
$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

$ source ~/.bashrc

$ k get nodes -o wide
```

### 2. Install rancher

- Cert-Manager와 Rancher를 Helm을 이용하여 설치합니다.
- 브라우저로 https:/rancher.kw01에 접속하여 암호를 설정합니다.

```bash
# Install cert-manager
$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.10.0/cert-manager.yaml

$ kn cert-manager
$ k rollout status deploy

# Install Rancher
$ helm repo add rancher-latest https://releases.rancher.com/server-charts/latest   
$ helm install rancher rancher-latest/rancher \
  --set hostname=rancher.kw01 \
  --set bootstrapPassword=admin \
  --set replicas=1 \
  --set global.cattle.psp.enabled=false \
  --create-namespace -n cattle-system

# https://rancher.kw01
# bootstrap passwd : admin & change passwd
```

---

### 3. Install gitlab

```bash
# Install local-path storage class & set default
$ kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.22/deploy/local-path-storage.yaml
$ kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Install gitlab
$ helm repo add gitlab https://charts.gitlab.io/
  
$ helm upgrade -i gitlab gitlab/gitlab \
--set global.edition=ce \
--set global.hosts.domain=kw01 \
--set global.ingress.configureCertmanager=false \
--set global.ingress.class=nginx \
--set certmanager.install=false \
--set nginx-ingress.enabled=false \
--set gitlab-runner.install=false \
--set prometheus.install=false \
-n gitlab --create-namespace

# get root initial password
$ k get -n gitlab secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 -d

# login gitlab.kw01 as root / %YOUR_INITIAL_PASSWORD%
# https://gitlab.kw01/admin/application_settings/general > visibility & access controls > import sources > Repository By URL

# create User with YOUR ID / PASSWD
# argo / abcd!234 argo@devops

# approve YOUR ID with root account admin menu
# Login root and approve argo account

# Import source / deploy repository from gitlab
# Login argo and import projects
- https://github.com/flytux/kw-mvn : Project Name > KW-MVN
- https://github.com/flytux/kw-mvn-deploy : Project Name > KW-MVN-DEPLOY

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
  - host: gitlab.kw01
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
    - gitlab.kw01
    secretName: gitlab-web-tls
EOF


# Add selfsigned CA crt to gitlab runner via secret
# add to /etc/hosts 
cat << EOF | sudo tee -a /etc/hosts
10.128.15.215 gitlab.kw01
EOF

$ openssl s_client -showcerts -connect gitlab.kw01:443 -servername gitlab.kw01 < /dev/null 2>/dev/null | openssl x509 -outform PEM > gitlab.kw01.crt
# Custom CA 인증서를 추가합니다.
$ cat ca.crt >> gitlab.kw01.crt
$ k create secret generic gitlab-runner-tls --from-file=gitlab.kw01.crt  -n gitlab

# add in cluster dns gitlab.kw01 to coredns
$ k edit cm -n kube-system coredns

data:
  Corefile: |-
    .:53 {
        errors
        health  {
            lameduck 5s
        }
     hosts {
     10.128.15.215 gitlab.kw01
     fallthrough
     }
     ready
        kubernetes   cluster.local  cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus   0.0.0.0:9153
        forward   . /etc/resolv.conf
        cache   30
        loop
        reload
        loadbalance
    }
    
$ k run -it --rm curl --image curlimages/curl -- sh
/ $ ping gitlab.kw01

```
---

### 4. Install gitlab-runner

```bash
# Setup runner and get runner token from KW-MVN project

# https://gitlab.kw01/argo/kw-mvn/-/runners/new

# Configuration > Run untagged jobs 체크 > Submit
# Copy token glrt-wb_BLETYwEdVpP6qCyQX

$ cat << EOF > gitlab-runner-values.yaml
gitlabUrl: http://gitlab.kw01

runnerToken: glrt-wb_BLETYwEdVpP6qCyQX
rbac:
  create: true

certsSecretName: gitlab-runner-tls

runners:
  config: |
    [[runners]]
      [runners.kubernetes]
        namespace = "{{.Release.Namespace}}"
        image = "ubuntu:16.04"
    [[runners.kubernetes.volumes.pvc]]
      mount_path = "/cache/maven.repository"
      name = "gitlab-runner-cache-pvc"
EOF

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

# Gitlab Runner 설치
$ helm upgrade -i gitlab-runner -f gitlab-runner-values.yaml gitlab/gitlab-runner
```  
---

### 4. install argocd & docker registry

```bash
# install argocd
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# add argocd ssl-passthrough args to ingress-controller
$ k edit ds -n kube-system rke2-ingress-nginx-controller

# add "--enable-ssl-passthrough" at line 53
  - --watch-ingress-without-class=true
  - --enable-ssl-passthrough
# save and qute (:wq)

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
  - host: argocd.kw01
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
- https://argocd.kw01/settings/certs?addTLSCert=true
- add name gitlab.kw01 & paste gitlab.kw01.crt pem file

$ cat gitlab.kw01.crt

# add argocd app 

$ k exec -it $(k get pods -l app.kubernetes.io/name=argocd-server -o name) bash

# check argocd user id and password
$ argocd login argocd-server.argocd --insecure --username admin --password e3m7VS-JpcpczVcq
$ argocd repo add http://gitlab.kw01/argo/kw-mvn-deploy.git --username argo --insecure-skip-server-verification
# enter gitlab password : abcd!234

$ kubectl -n argocd apply -f - <<"EOF"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kw-mvn
spec:
  destination:
    name: ''
    namespace: deploy
    server: 'https://kubernetes.default.svc'
  source:
    path: dev
    repoURL: 'https://gitlab.kw01/argo/kw-mvn-deploy.git'
    targetRevision: kust
  sources: []
  project: default
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
EOF

# Install docker registry
$ helm repo add twuni https://helm.twun.io
$ helm fetch twuni/docker-registry

$ cat << EOF >> values.yaml
service:
  name: registry
  type: NodePort
  port: 5000
  nodePort: 30005
persistence:
  accessMode: 'ReadWriteOnce'
  enabled: true
  size: 10Gi
  storageClass: 'local-path'
EOF

$ helm install docker-registry -f values.yaml docker-registry-2.2.2.tgz -n registry --create-namespace

$ curl -v $MY_NODE1_IP:30005/v2/_catalog

# nerdctl download
$ wget https://github.com/containerd/nerdctl/releases/download/v1.3.1/nerdctl-full-1.3.1-linux-amd64.tar.gz
$ sudo tar Cxzvvf /usr/local nerdctl-full-1.3.1-linux-amd64.tar.gz

# nerdctl 설정
$ sudo mkdir -p /etc/nerdctl
$ cat << EOF | sudo tee /etc/nerdctl/nerdctl.toml
debug          = false
debug_full     = false
address        = "unix:///run/k3s/containerd/containerd.sock"
namespace      = "k8s.io"
cgroup_manager = "cgroupfs"
hosts_dir      = ["/etc/containerd/certs.d", "/etc/docker/certs.d"]
EOF

# admin / 1 로 로그인
$ sudo nerdctl --insecure-registry login $MY_NODE1_IP:30005 # 노드IP 값으로 변경

# 컨테이너 런타임에 Private Registry 인증 / insecure 설정
# 레지스트리 주소를 자신의 주소로 변경합니다.
$ cat << EOF | sudo tee /etc/rancher/rke2/registries.yaml
mirrors:
  $MY_NODE1_IP:30005:
    endpoint:
      - http://10.128.15.213:30005
configs:
  $MY_NODE1_IP:30005:
    auth:
      username: admin 
      password: 1 
    tls:
      insecure_skip_verify: true
EOF

$ sudo systemctl restart rke2-server

# 워커노드도 동일하게 적용해 줍니다.
# 워커노드 로그인 후
$ cat << EOF | sudo tee /etc/rancher/rke2/registries.yaml
mirrors:
  $MY_NODE1_IP:30005:
    endpoint:
      - http://10.128.15.213:30005
configs:
  $MY_NODE1_IP:30005:
    auth:
      username: admin 
      password: 1 
    tls:
      insecure_skip_verify: true
EOF

$ sudo systemctl restart rke2-agent


# 아래 파일에 insecure 및 인증 설정 추가 확인 
$ sudo cat /var/lib/rancher/rke2/agent/etc/containerd/config.toml

### 5. develop build script

- https://gitlab.kw01/argo/kw-mvn/-/ci/editor?branch_name=main

```bash

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=/cache/maven.repository"
  IMAGE_URL: "10.128.15.213:30005/kw-mvn"
  DEPLOY_REPO_URL: "https://gitlab.kw01/argo/kw-mvn-deploy.git"
  DEPLOY_REPO_CREDENTIALS: "https://argo:abcd!234@gitlab.vm01/jaehoon/kw-mvn-deploy.git"
  REGISTRY_USER_ID: "admin"
  REGISTRY_USER_PASSWORD: "1"
  ARGO_URL: "argocd-server.argocd"
  ARGO_USER_ID: "admin"
  ARGO_USER_PASSWORD: "e3m7VS-JpcpczVcq"
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
    - echo "NEW_TAG=$IMAGE_URL:$COMMIT_TIME-$CI_JOB_ID" >> build.env
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
    - sed -i "s|newTag:.*$|newTag: $NEW_TAG|" dev/kustomization.yaml
    - cat dev/kustomization.yaml
    
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

