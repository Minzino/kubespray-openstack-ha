# OpenStack HA Kubernetes 배포 가이드

OpenStack 환경에서 Kubespray v2.26.0과 Kubernetes 1.30.8을 사용하여 고가용성 Kubernetes 클러스터를 배포하는 가이드입니다.

## 클러스터 구성

### 아키텍처
- **3개 마스터 노드**: Control plane + stacked etcd
- **3개 워커 노드**: 애플리케이션 워크로드 노드  
- **네트워크 플러그인**: Cilium (eBPF 기능 지원)
- **고가용성**: 3개 제어 플레인 노드

### 노드 구성
```
마스터 노드 (Control Plane):
├── master1 (10.0.0.10) - etcd1, kube-apiserver, kube-controller-manager, kube-scheduler
├── master2 (10.0.0.11) - etcd2, kube-apiserver, kube-controller-manager, kube-scheduler  
└── master3 (10.0.0.12) - etcd3, kube-apiserver, kube-controller-manager, kube-scheduler

워커 노드:
├── worker1 (10.0.0.20) - kubelet, kube-proxy, cilium-agent
├── worker2 (10.0.0.21) - kubelet, kube-proxy, cilium-agent
└── worker3 (10.0.0.22) - kubelet, kube-proxy, cilium-agent
```

## 사전 준비사항

### OpenStack 인프라
1. **OpenStack 환경** 구성 요소:
   - Nova (Compute)
   - Neutron (Networking) 
   - Cinder (Block Storage) - 선택사항
   - Heat (Orchestration) - 선택사항

2. **VM 인스턴스** 사양:
   - **마스터 노드**: 4 vCPU, 8GB RAM, 50GB 디스크
   - **워커 노드**: 2 vCPU, 4GB RAM, 50GB 디스크
   - SSH 접속 설정 완료
   - 보안 그룹에서 필요한 포트 허용

3. **네트워크 설정**:
   - 사설 네트워크 및 서브넷 (예: 10.0.0.0/24)
   - 외부 게이트웨이가 연결된 라우터
   - Floating IP (선택사항)

### 필요 포트
```
마스터 노드:
- 6443/tcp  - Kubernetes API server
- 2379-2380/tcp - etcd server client API
- 10250/tcp - Kubelet API

워커 노드:
- 10250/tcp - Kubelet API  
- 30000-32767/tcp - NodePort Services

모든 노드:
- 22/tcp - SSH
- 179/tcp - BGP (Cilium)
- 4789/udp - VXLAN (Cilium)
```

## 1단계: OpenStack 인스턴스 준비

### 보안 그룹 생성
```bash
# Kubernetes 클러스터용 보안 그룹 생성
openstack security group create k8s-cluster

# SSH 허용
openstack security group rule create --protocol tcp --dst-port 22 k8s-cluster

# Kubernetes API 허용
openstack security group rule create --protocol tcp --dst-port 6443 k8s-cluster

# etcd 허용  
openstack security group rule create --protocol tcp --dst-port 2379:2380 k8s-cluster

# Kubelet 허용
openstack security group rule create --protocol tcp --dst-port 10250 k8s-cluster

# NodePorts 허용
openstack security group rule create --protocol tcp --dst-port 30000:32767 k8s-cluster

# Cilium BGP 허용
openstack security group rule create --protocol tcp --dst-port 179 k8s-cluster

# Cilium VXLAN 허용
openstack security group rule create --protocol udp --dst-port 4789 k8s-cluster

# 내부 통신 허용
openstack security group rule create --protocol icmp k8s-cluster
openstack security group rule create --remote-group k8s-cluster k8s-cluster
```

### 인스턴스 생성
```bash
# 마스터 노드 생성
for i in {1..3}; do
  openstack server create \
    --flavor m1.large \
    --image ubuntu-20.04 \
    --key-name your-keypair \
    --security-group k8s-cluster \
    --network private-network \
    master${i}
done

# 워커 노드 생성
for i in {1..3}; do
  openstack server create \
    --flavor m1.medium \
    --image ubuntu-20.04 \
    --key-name your-keypair \
    --security-group k8s-cluster \
    --network private-network \
    worker${i}
done
```

## 2단계: Kubespray 준비

### 클론 및 설정
```bash
# 이 저장소 클론
git clone <repository-url>
cd kubespray

# Kubernetes 1.30 호환 버전으로 체크아웃
git checkout v2.26.0

# Python 의존성 설치
pip install -r requirements.txt
```

### 인벤토리 설정
```bash
# 사전 구성된 OpenStack HA 인벤토리 복사
cp -r inventory/openstack-ha inventory/my-openstack-cluster

# 실제 IP 주소로 업데이트
vim inventory/my-openstack-cluster/inventory.ini
```

인벤토리 파일에서 IP 주소 업데이트:
```ini
[all]
master1 ansible_host=<MASTER1_IP> ip=<MASTER1_IP> etcd_member_name=etcd1
master2 ansible_host=<MASTER2_IP> ip=<MASTER2_IP> etcd_member_name=etcd2
master3 ansible_host=<MASTER3_IP> ip=<MASTER3_IP> etcd_member_name=etcd3
worker1 ansible_host=<WORKER1_IP> ip=<WORKER1_IP>
worker2 ansible_host=<WORKER2_IP> ip=<WORKER2_IP>
worker3 ansible_host=<WORKER3_IP> ip=<WORKER3_IP>
```

## 3단계: Kubernetes 클러스터 배포

### 배포 전 점검
```bash
# 모든 노드 연결 테스트
ansible -i inventory/my-openstack-cluster/inventory.ini all -m ping

# 디스크 공간 확인 (각 노드 50GB 이상 필요)
ansible -i inventory/my-openstack-cluster/inventory.ini all -m shell -a "df -h /"
```

### 클러스터 배포
```bash
# 클러스터 배포 (15-30분 소요)
ansible-playbook -i inventory/my-openstack-cluster/inventory.ini \
  --become --become-user=root \
  cluster.yml
```

### 배포 확인
```bash
# 첫 번째 마스터 노드에 SSH 접속
ssh ubuntu@<MASTER1_IP>

# 클러스터 상태 확인
sudo kubectl get nodes
sudo kubectl get pods --all-namespaces

# Cilium 실행 확인
sudo kubectl get pods -n kube-system -l k8s-app=cilium
```

## 4단계: 클러스터 접근

### kubectl 접근 설정
```bash
# 마스터 노드에서 kubeconfig 복사
scp ubuntu@<MASTER1_IP>:~/.kube/config ~/.kube/config-openstack-ha

# KUBECONFIG 환경변수 설정
export KUBECONFIG=~/.kube/config-openstack-ha

# 접근 테스트
kubectl get nodes
kubectl cluster-info
```

## 문제해결

### 일반적인 문제

1. **SSH 연결 실패**
   ```bash
   # 보안 그룹과 키 페어 확인
   openstack server show <instance-name>
   ```

2. **디스크 공간 문제**
   ```bash
   # 모든 노드의 디스크 사용량 확인
   ansible -i inventory/my-openstack-cluster/inventory.ini all -m shell -a "df -h"
   ```

3. **네트워크 연결 문제**
   ```bash
   # 노드 간 내부 연결 테스트
   ansible -i inventory/my-openstack-cluster/inventory.ini all -m shell -a "ping -c 3 <MASTER1_IP>"
   ```

### 클러스터 초기화
문제 발생 시 클러스터를 완전히 초기화하려면:
```bash
# 클러스터 초기화 (경고: 모든 데이터가 삭제됩니다!)
ansible-playbook -i inventory/my-openstack-cluster/inventory.ini \
  --become --become-user=root \
  reset.yml
```

## 구성 세부사항

- **Kubespray 버전**: v2.26.0
- **Kubernetes 버전**: v1.30.8  
- **컨테이너 런타임**: containerd
- **네트워크 플러그인**: Cilium v1.15.4
- **DNS**: CoreDNS

고급 구성 옵션은 `inventory/openstack-ha/group_vars/` 디렉토리의 파일들을 참조하세요.