# SA사업그룹 Rancher/Harvester 기본 가이드 V1.0  
2025.05.11

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

---

## 2. Harvester 설치

### 2.1 VMware 설정
- [Harvester ISO 다운로드](https://releases.rancher.com/harvester/v1.5.0/harvester-v1.5.0-amd64.iso)

주의사항
- VirtualBox 미지원
- 최소 12GB RAM 필요
- VMware에서 VM 생성 시 사양 참고

### 2.2 Harvester 설치
![그림 1. Harvester 초기 설치 화면_01](https://github.com/user-attachments/assets/e76bac57-447c-4743-bc4c-df42548cb314)
[ 그림 1. Havester 초기 설치 화면_01]

## 3. Rancher 설치

### 3.1 RKE2 설치 (Master)
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
