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
