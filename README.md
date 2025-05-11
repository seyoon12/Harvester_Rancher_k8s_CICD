#  Rancher/Harvester 설치 및 Fleet CICD

---

## 목차
1. 개요
2. Harvester 설치
3. Rancher 설치
4. Rancher와 Harvester 연동
5. Rancher CI/CD
6. Jenkins 설정
7. Gitea 설정
8. Fleet 설정

---

## 1. 개요

Rancher와 Harvester 설치 및 연동, Fleet을 활용한 GitOps 기반 CI/CD 구성 가이드입니다.  
테스트 환경은 VMware 기반이며, 모든 구성 요소는 오픈소스 도구를 활용했습니다.

### 1.1 기술 가이드 소개
- Harvester: 오픈소스 HCI 플랫폼
- Rancher: Kubernetes 통합 관리 플랫폼
- Jenkins: CI 도구
- Harbor: 이미지 레지스트리
- Gitea: 경량 Git 서버
- Fleet: Rancher GitOps 배포 도구

### 1.2 서비스 구성 요소
- Harvester (v1.5.0)
- Rancher (helm chart stable)
- Jenkins (lts-jdk17)
- Harbor (helm chart)
- Gitea (latest)
- Fleet (내장)

### 1.3 구성 사항
- VMware Workstation 17 Pro  
- Harvester ISO: v1.5.0  
- VM 스펙: CPU 4, MEM 8500MB, Disk 250GB  
- Rancher VM 스펙: CPU 4, MEM 4192MB
- 
### 2.1 VMware 설정
- [Harvester ISO 다운로드](https://releases.rancher.com/harvester/v1.5.0/harvester-v1.5.0-amd64.iso)
주의사항
- VirtualBox 미지원
- 최소 12GB RAM 필요
- VMware에서 VM 생성 시 사양 참고

---

## 2. Harvester 설치

### 그림 1. Harvester 초기 설치 화면_01

![그림 1. Harvester 초기 설치 화면_01](https://github.com/user-attachments/assets/e76bac57-447c-4743-bc4c-df42548cb314)

### 그림 2. Harvester 초기 설치 화면_02

![image](https://github.com/user-attachments/assets/ebed868e-e7e1-411c-938f-49a6af3ba5d2)

### 그림 3. Harvester 초기 설치 화면_03
![image](https://github.com/user-attachments/assets/41d437f8-eba8-4ebe-adb0-180e0d24b4c1)

### 그림 4. Harvester 초기 설치 화면_04
![image](https://github.com/user-attachments/assets/41d437f8-eba8-4ebe-adb0-180e0d24b4c1)

### 그림 5. Harvester 초기 설치 화면_05
![image](https://github.com/user-attachments/assets/3d972742-310b-487a-90d4-e737889b8b09)

### 그림 6. Harvester 초기 설치 화면_06
![image](https://github.com/user-attachments/assets/a29347af-9f2f-4194-8a35-f98acdec7965)


## 3. Rancher 설치

### 3.1 RKE2 설치 (Master)
개인 컴퓨터 메모리가 16Gb이하일 경우 
1. Rancher 4gb, Harvester 8을 주기 때문에 Harvester에서 OOM이 날 가능성이 있어 저는 Harvester는 VM,
Rancher는 클라우드에 올려 연동하였습니다.

```bash
hostnamectl set-hostname master
swapoff -a
systemctl disable --now ufw
systemctl disable --now apparmor.service
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh -
systemctl enable rke2-server
systemctl restart rke2-server
mkdir -p ~/.kube
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
```

```bash
# Master vm
hostnamectl set-hostname master
swapoff -a
sudo systemctl disable --now ufw
sudo systemctl disable --now apparmor.service
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="server" sh –
systemctl enable rke2-server
systemctl restart rke2-server
systemctl status rke2-server
mkdir ~/.kube/
cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
export PATH=$PATH:/var/lib/rancher/rke2/bin/
echo 'export PATH=/usr/local/bin:/var/lib/rancher/rke2/bin:$PATH' >> ~/.bashrc
echo 'alias k="kubectl"' >> ~/.bashrc
echo 'alias kns="kubectl config set-context --current --namespace"' >> ~/.bashrc
source ~/.bashrc
```

```bash
# Slave VM
hostnamectl set-hostname slave01
swapoff -a
sudo systemctl disable --now ufw
sudo systemctl disable --now apparmor.service
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh –
mkdir -p /etc/rancher/rke2/
vi /etc/rancher/rke2/config.yaml
# Master vm의 cat /var/lib/rancher/rke2/server/node-token값 복사 붙여넣은 후 systemctl restart rke2-agent
```

```bash
# 예시
server: https://10.0.2.15:9345
token: K10e133e33123a042c358b3db0e92ff14a7c61dd77fd106438bb7f9355d0100c0ce::server:0ccc5d476c0f508c04f59ec0e7163755
```

```bash
# OPTION 01 - Rancher를 VM에 설치 시
# Master VM
# cert-manager 설치
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl get pods --namespace cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook

# helm 설치
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# rancher 설치
kubectl create namespace cattle-system
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
helm search repo rancher-stable
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.test.com \ 
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=wntpqhd1326@gmail.com \
  --set letsEncrypt.ingress.class=nginx
```

```bash
# OPTION 02 - Rancher를 Cloud에 설치 시
# Master VM
# cert-manager 설치
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
kubectl -n cert-manager rollout status deploy/cert-manager
kubectl get pods --namespace cert-manager
kubectl -n cert-manager rollout status deploy/cert-manager-webhook

# helm 설치
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# domain 설정 (무료)
https://www.duckdns.org/
```
### 그림 7. duckdns 화면
![image](https://github.com/user-attachments/assets/c057573e-d313-474a-bcb1-4bae37d833a9)

```bash
server {
    listen 80;
    server_name DuckDNS에서 설정한 URL

    root /var/www/html;
    index index.html;

    location /.well-known/acme-challenge/ {
        allow all;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```bash
certbot --nginx -d DuckDNS에서 설정한 URL

kubectl -n cattle-system create secret generic tls-rancher-ingress \
  --from-file=tls.crt=/etc/letsencrypt/live/ DuckDNS에서 설정한 URL /fullchain.pem \
  --from-file=tls.key=/etc/letsencrypt/live/ DuckDNS에서 설정한 URL /privkey.pem \
  --from-file=ca.crt=/etc/letsencrypt/live/ DuckDNS에서 설정한 URL /chain.pem

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=DuckDNS에서 설정한 URL \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=secret
```

### 그림 8. Rancher 기본 대시보드
![image](https://github.com/user-attachments/assets/e25e3b3c-77de-4c0f-959d-77567f3ca581)

### 그림 9. Rancher Harvester UI 설치 
![image](https://github.com/user-attachments/assets/998d6ba9-e05a-4429-be26-2273251fc021)

### 그림 10. Rancher의 Harvester Registration URL 저장
![image](https://github.com/user-attachments/assets/6a23d5db-dd28-4244-8afc-2214d7f527db)

### 그림 11. Harvester 에서 해당 Registration URL 복사 후 붙여넣기
![image](https://github.com/user-attachments/assets/e384c64e-a173-475c-8ac9-200371bcdf8f)

### 그림 12. Harvester ssh에서 rancher 연동_01
![image](https://github.com/user-attachments/assets/c62115cc-9897-41d6-b6c6-834c21976518)

### 그림 13. Harvester ssh에서 rancher 연동_02
![image](https://github.com/user-attachments/assets/ea92f632-bf70-408c-921a-10d8e8c10133)

### 그림 13. Harvester ssh에서 rancher 연동_03
![image](https://github.com/user-attachments/assets/5cf2bc4a-59d0-48cf-9500-5677ba2d2e08)

## 3. CICD
### 3.1 설치
```bash
# longhorn 설치
# SlaveVM
apt update -y
apt install nfs-common
echo "iscsi_tcp" | sudo tee -a /etc/modules
apt-get install open-iscsi
modprobe iscsi_tcp

# MasterVM
apt update -y
apt install nfs-common
echo "iscsi_tcp" | sudo tee -a /etc/modules
apt-get install open-iscsi
modprobe iscsi_tcp
helm repo add longhorn https://charts.longhorn.io
helm repo update
k create ns longhorn-system
helm install longhorn longhorn/longhorn \
--namespace longhorn-system \
--version 1.6.4 \
--set defaultSettings.defaultDataPath="/var/lib/longhorn" \
--set defaultSettings.defaultDataLocality="best-effort" \
--set defaultSettings.replicaAutoBalance="best-effort" \
--set defaultSettings.defaultReplicaCount=2 \
--set defaultSettings.defaultLonghornStaticStorageClass="longhorn-static"
k get sc
```

```bash
# harbor 설치
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
helm repo add harbor https://helm.goharbor.io
helm repo update
helm install harbor harbor/harbor \
  --namespace cicd --create-namespace \
  --set expose.type=nodePort \
  --set expose.nodePort.ports.http.nodePort=30003 \
  --set expose.tls.enabled=false \
  --set externalURL=http://192.168.32.10:30003 \

# Harbor기본 비밀번호
ID : admin
PW : Harbor12345
# docker http 통신 설정 및 docker config secret 생성
mkdir -p /etc/docker
echo '{ "insecure-registries":["your_ip:30003"] }' | sudo tee /etc/docker/daemon.json 
systemctl restart docker

docker login your_ip:30003 -u 'your_harbor_id'

kubectl -n cicd create secret generic kaniko-docker-config \
  --from-file=/root/.docker/config.json
```

```bash
# Jenkins pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: cicd
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: longhorn

# Jenkins sa
kubectl create serviceaccount jenkins -n cicd

kubectl create clusterrolebinding jenkins-sa-cluster-admin \
  --clusterrole=cluster-admin \
--serviceaccount=cicd:jenkins

# Jenkins secret
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-sa-token
  namespace: cicd
  annotations:
    kubernetes.io/service-account.name: jenkins
type: kubernetes.io/service-account-token

# Jenkins
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: cicd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      securityContext:
        fsGroup: 1000
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-jdk17
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
            claimName: jenkins-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: jenkinsservice
  namespace: cicd
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      nodePort: 30088
    - name: agent
      port: 50000
      targetPort: 50000
      nodePort: 30089   # jenkins 터널 용도 / 임의의 숫자 넣었음

```

```bash
# gitea pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-pvc
  namespace: cicd
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: longhorn

# gitea
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea
  namespace: cicd
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      containers:
        - name: gitea
          image: gitea/gitea:latest
          ports:
            - containerPort: 3000
            - containerPort: 22
          volumeMounts:
            - name: gitea-data
              mountPath: /data
      volumes:
        - name: gitea-data
          persistentVolumeClaim:
            claimName: gitea-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: gitea-service
  namespace: cicd
spec:
  type: NodePort
  selector:
    app: gitea
  ports:
    - name: http
      port: 3000
      targetPort: 3000
      nodePort: 32000
```

```bash
# rke2 설정
# slave
vi /etc/rancher/rke2/registries.yaml

mirrors:
  "192.168.32.10:30003":
    endpoint:
      - http://192.168.32.10:30003

# master
vi /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."192.168.32.10:30003"]
  endpoint = ["http://192.168.32.10:30003"]
```
### 3.2 Jenkins 설정
### 그림 14. Jenkins cloud 설정_01 / Kubernetes server certificate key
![image](https://github.com/user-attachments/assets/b3ac38ff-b8f2-4e91-9dca-f7d9363356de)

```bash
# jenkins Kubernetes server certificate key
kubectl -n cicd get secret jenkins-sa-token -o yaml
```

### 그림 15. Jenkins cloud 설정_02 / Credentials
![image](https://github.com/user-attachments/assets/796255f7-35dd-4df0-8be5-dcdaa8f86a5f)

```bash
# jenkins cloud credentials
kubectl -n cicd get secret jenkins-sa-token -o jsonpath='{.data.token}' | base64 -d
```

### 그림 16. Jenkins cloud 설정_03 / Jenkins URL, Tunnel
![image](https://github.com/user-attachments/assets/3f8b33a9-5d1a-4fd0-ba3a-fe5805e798e4)

### 그림 17. Jenkins cloud 설정_04 / pod_template
![image](https://github.com/user-attachments/assets/20116913-38fb-4164-b313-0ee9a6782419)

### 3.3 Gitea repo

### 그림 18. Gitea repo
![image](https://github.com/user-attachments/assets/b44ed2a2-afd0-43b8-ac58-e6ff0d03bd97)

```bash
pipeline {
  agent {
    kubernetes {
      yaml """
apiVersion: v1
kind: Pod
metadata:
  name: kaniko-build-pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - /busybox/cat
    tty: true
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
  - name: git
    image: alpine/git
    command:
    - cat
    tty: true
  - name: jnlp
    image: jenkins/inbound-agent:latest
  volumes:
  - name: docker-config
    secret:
      secretName: kaniko-docker-config
"""
      defaultContainer 'kaniko'
    }
  }

  environment {
    IMAGE_PUSH_DESTINATION = "192.168.32.10:30003/seyoon/app:${BUILD_NUMBER}"
  }

  stages {
    stage('Build & Push with Kaniko') {
      steps {
        container('kaniko') {
          sh '''
            /kaniko/executor --dockerfile=Dockerfile \
                             --context=$(pwd) \
                             --destination=$IMAGE_PUSH_DESTINATION \
                             --insecure \
                             --skip-tls-verify \
                             --insecure-pull

            sed -i "s|image: 192.168.32.10:30003/seyoon/app:[^[:space:]]*|image: 192.168.32.10:30003/seyoon/app:${BUILD_NUMBER}|" test-deployment.yaml
          '''
        }
      }
    }

    stage('Git Push Updated Deployment') {
      steps {
        container('git') {
          withCredentials([usernamePassword(credentialsId: 'gitea_token', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
            sh '''
              git config --global user.email "jenkins@local.com"
              git config --global user.name "Jenkins CI"
              git config --global --add safe.directory /home/jenkins/agent/workspace/seyoon_main
              git checkout main
              git remote set-url origin http://${GIT_USER}:${GIT_TOKEN}@192.168.32.10:32000/admin/seyoon.git
              git add test-deployment.yaml
              git commit -m "Update image tag to ${BUILD_NUMBER}"
              git push origin main
            '''
          }
        }
      }
    }
  }
}
```

###
![image](https://github.com/user-attachments/assets/dff630e1-b812-47c0-9804-93f1e7c27791)












