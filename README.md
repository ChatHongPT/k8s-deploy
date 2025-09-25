# Kubernetes 클러스터 구축

<img width="1048" height="464" alt="image" src="https://github.com/user-attachments/assets/a096b2dd-831e-4286-9349-cf48452fa20e" />

### 클러스터 구성
<img width="1324" height="524" alt="제목 없는 다이어그램 drawio (2)" src="https://github.com/user-attachments/assets/7f5347f8-4cfc-4e37-90d3-d03a78078f58" />

- **Master Node**: myserver00 (10.0.2.15)
- **Worker Nodes**: myserver01 (10.0.2.20), myserver02 (10.0.2.25)
- **Container Runtime**: containerd
- **Network Plugin**: Calico
- **Kubernetes Version**: v1.30.14

---

### VirtualBox의 다중 가상 머신 통신을 위한 설정

#### 상단 메뉴에서 파일 → 도구 → 네트워크 관리자로 이동 / NAT 네트워크 탭에서 추가(+) 버튼을 눌러 새로운 NAT 네트워크 생성
<img width="990" height="594" alt="image" src="https://github.com/user-attachments/assets/ea09f197-4c23-4024-a054-2750060e1a1d" />

#### NAT 네트워크 설정 → 포트 포워딩에서 규칙 추가
<img width="839" height="245" alt="image" src="https://github.com/user-attachments/assets/a80c7b45-cefa-4781-af1c-56b4ce1ab001" />

#### 고정 IP설정하기 
<img width="1108" height="321" alt="image" src="https://github.com/user-attachments/assets/d01dbb9c-d154-410b-8dcf-a52a189959d2" />

- netplan 설정 파일 확인 & 편집
- 00-installer-config.yaml  또는 01-netcfg.yaml 등의 파일 생성 
- 설정 파일 편집
  - 설정 파일의 형식은 YAML
  - DHCP(Dynamic Host Configuration Protocol, 자동 IP 할당 시스템)를 사용하지 않고 고정 IP(Static IP)를 설정하려면, 다음과 같이 수정해야 함

```bash
sudo ls /etc/netplan
sudo nano /etc/netplan/01-netcfg.yaml  
sudo rm /etc/netplan/50-cloud-init.yaml 
```

#### 01-netcfg.yaml 파일

```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:  # 본인의 실제 인터페이스 이름을 확인 후 변경
      dhcp4: no  # DHCP 비활성화
      addresses:
        - 10.0.2.15/24  # 원하는 고정 IP + 서브넷
      routes:
        - to: default
          via: 10.0.2.1  # 기본 게이트웨이 (라우터 주소)
      nameservers:
        addresses:
          - 8.8.8.8  # Google DNS
          - 1.1.1.1  # Cloudflare DNS
```

#### 설정 후 적용
<img width="721" height="63" alt="image" src="https://github.com/user-attachments/assets/a655b044-d102-43ba-9fea-fc35c92857c3" />

```bash
# 소유자가 root인지 확인
ls -l /etc/netplan/01-netcfg.yaml
sudo netplan apply

# 권한을 600으로 변경 (root만 읽기/쓰기 가능) 
sudo chmod 600 /etc/netplan/01-netcfg.yaml

ls -l /etc/netplan/01-netcfg.yaml
```
#### Host 이름 변경 및 Host이름으로 접속을 위한 설정

```bash
sudo hostnamectl set-hostname myserver02

# 변경된 host이름 확인
cat /etc/hostname

# 실제 적용을 위한 재부팅
sudo init 6
```

#### 직접 수정해야 할 파일 
<img width="263" height="85" alt="image" src="https://github.com/user-attachments/assets/d87ec7d0-02b2-4e06-bcc1-cf53d69657e4" />

```bash
myserver01에 myserver02에게 host name으로 접속등 가능하게 IP/host이름 매핑해서 등록
sudo nano /etc/hosts

127.0.0.1 localhost
10.0.2.15 myserver01
10.0.2.20 myserver02   # 추가 등록시 id@hostname으로 접속 가능한 설정
```

#### swap 메모리 무효화 → 종류 → myserver02와 03 복제 

```bash
sudo -i
swapoff --all
```

#### Linux 서버간  통신 및 공유를 위한 SSH key 

1) SSH key 생성
파일을 전송할 로컬 서버(파일을 전송하는 서버)에서 SSH 키를 생성

```bash
ssh-keygen -t rsa -b 4096 
```
​
#### 개인키와 공개키 파일 생성

1) 개인키  파일명 : ~/.ssh/id_rsa
2) 공개키  파일명 : ~/.ssh/id_rsa.pub
-> 비밀번호는 필요 시 입력할 수 있지만, 비워 두면 더 편리

```bash
key 생성 후 확인 명령어
ls -l .ssh

cat .ssh/authorized_keys
cat .ssh/id_rsa
cat .ssh/id_rsa.pub
```
---

#### 원격 서버에 SSH 공개 키 추가

SSH 키를 통해 비밀번호 없이 접속할 수 있도록 로컬 서버의 공개 키를 원격 서버에 추가함

공개 키를 원격 서버의 **~/.ssh/authorized_keys 파일에 추가**

```bash
**# 01
ssh-copy-id ubuntu@myserver02
ssh-copy-id ubuntu@myserver03

# 02
ssh-copy-id ubuntu@myserver01
ssh-copy-id ubuntu@myserver03

# 03
ssh-copy-id ubuntu@myserver01
ssh-copy-id ubuntu@myserver02**
```
---
####  br_netfilter 모듈 활성화

Kubernetes Pod 네트워크가 브리지 네트워크를 사용하기 때문에 반드시 활성화해야 함

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

부팅 시 자동으로 로드되도록 하려면 /etc/modules-load.d/k8s.conf 파일을 생성

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
#### sysctl 설정 추가

/etc/sysctl.d/k8s.conf 파일에 다음 내용 추가

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

#### 설정 적용
   
#### sysctl 설정 즉시 적용
```bash
sudo sysctl --system
```
#### 전체 스크립트 (root 계정 실행)

1) root 계정 전환
```bash
sudo -i
```

2) 커널 모듈 설정
```bash
mkdir -p /etc/modules-load.d/
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

3) 모듈 즉시 로드
```bash
modprobe overlay
modprobe br_netfilter
```

4) sysctl 설정 파일 작성
```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```
5) sysctl 적용
```bash
sysctl --system
```
5) 모듈 로드 확인
```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```
→ 결과에 br_netfilter, overlay 가 보이면 정상.

6) sysctl 값 확인
```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
sysctl net.ipv4.ip_forward
```

→ 모두 = 1 이면 정상

---

## 3. 쿠버네티스 설치 (Ubuntu 24.04 기준)
<img width="635" height="635" alt="image" src="https://github.com/user-attachments/assets/90098f93-39d8-4e0c-8242-d5bdc17b0019" />

#### Containerd 설치 및 설정
```bash
sudo apt-get update
sudo apt-get install -y containerd
```

#### 버전 확인
```bash
containerd --version
systemctl status containerd
```
#### 설정 디렉토리 생성
```bash
sudo mkdir -p /etc/containerd
```
#### 기본 설정값 생성
```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```
systemd cgroup 활성화
### 방법 1: vi로 직접 수정
```bash
sudo vi /etc/containerd/config.toml
```
-> "SystemdCgroup = false" → "true" 로 변경

### 방법 2: sed로 한 줄 수정
```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```
#### 적용
```bash
sudo systemctl restart containerd
sudo systemctl status containerd
```
## Kubernetes 저장소 추가
#### 기존 설정 제거 (재설치 시 필요)
```bash
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo rm -f /etc/apt/trusted.gpg.d/kubernetes.gpg
```
#### 의존 패키지 설치
```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```
#### 키 저장 디렉토리 생성
```bash
sudo mkdir -p /etc/apt/keyrings
```
#### Kubernetes GPG 키 추가
```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
#### 저장소 추가
```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
```
#### 인증 옵션 허용 (Ubuntu 24.04 호환)
```bash
sudo tee /etc/apt/apt.conf.d/99allow-insecure-repositories <<EOF
APT::Get::AllowUnauthenticated "true";
Acquire::AllowInsecureRepositories "true";
Acquire::AllowDowngradeToInsecureRepositories "true";
EOF
```
#### 업데이트
```bash
sudo apt-get update --allow-unauthenticated
```
#### kubeadm / kubelet / kubectl 설치
```bash
sudo apt-get install -y --allow-unauthenticated kubelet kubeadm kubectl
```

#### 설치 확인
```bash
kubectl version --client --output=yaml
kubeadm version --output=yaml
kubelet --version
```
---

## 4. 마스터 노드 초기화 (myserver00에서만)

### 클러스터 초기화
```bash
sudo kubeadm init --apiserver-advertise-address=10.0.2.15 --pod-network-cidr=10.244.0.0/16
```

### kubectl 설정
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 조인 토큰 저장
초기화 완료 후 출력되는 join 명령어를 저장해두세요:
```bash
# 예시 출력:
# kubeadm join 10.0.2.15:6443 --token xxx --discovery-token-ca-cert-hash sha256:xxx
```
---

## 5. 네트워크 플러그인 설치 (Calico)

#### Calico란?
<img width="456" height="456" alt="image" src="https://github.com/user-attachments/assets/bb947835-3602-4b82-83be-58c6eb8d7d2d" />

> Calico는 Kubernetes 클러스터에서 가장 널리 사용되는 CNI(Container Network Interface) 플러그인 중 하나로, Pod 간 네트워크 통신과 네트워크 정책(NetworkPolicy)을 지원


#### Calico 매니페스트 다운로드 및 수정
```bash
curl -fsSLO https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml
```

#### Calico 설치
<img width="1104" height="517" alt="image" src="https://github.com/user-attachments/assets/131c632f-f535-4892-9471-5ca24e1a1fbe" />
```bash
kubectl apply -f calico.yaml
```

#### 설치 확인
<img width="781" height="133" alt="image" src="https://github.com/user-attachments/assets/f7a599da-adff-41e7-bd4f-aefd7f5d295f" />

```bash
# Calico Pod 상태 확인
kubectl get pods -n kube-system | grep calico

# 모든 시스템 Pod 확인
kubectl get pods -A
```

---

## 6. 워커 노드 조인

#### 각 워커 노드에서 실행
마스터 노드 초기화 시 출력된 명령어를 사용
```bash
sudo kubeadm join 10.0.2.15:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

#### 토큰이 만료된 경우 (마스터 노드에서)
```bash
# 새 토큰 생성
kubeadm token create --print-join-command
```

---

## 7. 클러스터 상태 확인
<img width="1099" height="220" alt="image" src="https://github.com/user-attachments/assets/9fa64d43-5662-4481-ba78-de26a3d7373f" />
#### 노드 상태 확인
```bash
kubectl get nodes -o wide
```

#### 예상 출력
```
NAME         STATUS   ROLES           AGE   VERSION
myserver00   Ready    control-plane   10m   v1.30.14
myserver01   Ready    <none>          8m    v1.30.14
myserver02   Ready    <none>          6m    v1.30.14
```

#### 모든 Pod 상태 확인
<img width="935" height="341" alt="image" src="https://github.com/user-attachments/assets/15e6db66-1fbd-4c1a-bfb2-bc4e886a102e" />
```bash
kubectl get pods -A
```

#### 클러스터 정보 확인
<img width="1040" height="126" alt="image" src="https://github.com/user-attachments/assets/8fea85cc-62ab-4a0f-84a0-1b5f39e9937f" />
```bash
kubectl cluster-info
```

---

## 8. 문제 해결

#### 노드가 NotReady인 경우

**원인 확인:**
```bash
# 노드 상세 정보 확인
kubectl describe node <NODE_NAME>

# kubelet 로그 확인 (해당 노드에서)
sudo journalctl -u kubelet -f

# CNI 설정 확인 (해당 노드에서)
ls -la /etc/cni/net.d/
```

**해결 방법:**
```bash
# kubelet 재시작
sudo systemctl restart kubelet

# containerd 재시작
sudo systemctl restart containerd
```

#### CoreDNS가 Pending인 경우

**원인 확인:**
```bash
# CoreDNS Pod 상세 확인
kubectl describe pods -n kube-system -l k8s-app=kube-dns
```

**해결 방법:**
```bash
# 마스터 노드 taint 제거 (single-master 환경)
kubectl taint nodes myserver00 node-role.kubernetes.io/control-plane:NoSchedule-

# CoreDNS 재시작
kubectl rollout restart deployment/coredns -n kube-system
```

### CNI 플러그인 초기화 실패

**에러 메시지:** `cni plugin not initialized`

**해결 방법:**
```bash
# 해당 노드에서 CNI 설정 확인
ls -la /etc/cni/net.d/

# calico-node 로그 확인
sudo crictl logs $(sudo crictl ps --name calico-node -q)

# 서비스 재시작
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

### 클러스터 완전 재초기화

**모든 노드에서:**
```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d/*
sudo rm -rf /var/lib/kubelet/
sudo rm -rf /var/lib/etcd/
```

**마스터에서 다시 초기화:**
```bash
sudo kubeadm init --apiserver-advertise-address=10.0.2.15 --pod-network-cidr=10.244.0.0/16
```

---

## 참고 자료

- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [Calico 네트워크 플러그인](https://docs.projectcalico.org/)
- [containerd 설정](https://containerd.io/docs/)
- [kubeadm 설치 가이드](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
