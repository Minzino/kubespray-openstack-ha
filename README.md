# Kubespray OpenStack HA Configuration

OpenStack 환경에서 고가용성(HA) Kubernetes 클러스터를 배포하기 위한 Kubespray 설정입니다.

## 프로젝트 구성

### 인프라 구성
- **총 7대 VM**: bastion(1) + master(3) + worker(3)
- **OS**: Ubuntu 22.04 LTS  
- **Kubernetes 버전**: 1.30.8
- **CNI**: Cilium (Helm으로 설치)
- **etcd**: 외부 데몬 방식 3중화
- **HA**: VIP 기반 HAProxy 로드밸런서

### 주요 기능
✅ **고가용성**: 3개 마스터 노드 + VIP  
✅ **Cilium CNI**: Helm 기반 설치  
✅ **Prometheus 메트릭**: 모든 K8s 컴포넌트에서 수집 가능  
✅ **외부 etcd**: Static Pod가 아닌 데몬 방식  
✅ **자동 Helm 설치**: Cilium 설치를 위한 Helm 자동 구성  

## 디렉토리 구조

```
├── README.md                    # 이 파일
└── inventory/
    └── openstack-ha/
        ├── inventory.ini        # 인벤토리 설정
        ├── DEPLOYMENT_GUIDE.md # 상세 배포 가이드
        └── group_vars/         # 모든 설정 변수들
            ├── all/            # 전역 설정
            └── k8s_cluster/    # Kubernetes 클러스터 설정
```

## 빠른 시작

### 1. 사전 준비
```bash
# OpenStack에 7대 VM 생성 (Ubuntu 22.04)
# bastion, master1-3, worker1-3

# Kubespray 클론
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray

# 이 설정을 kubespray에 복사
cp -r /path/to/this/repo/inventory/openstack-ha inventory/
```

### 2. 인벤토리 수정
`inventory/openstack-ha/inventory.ini`에서 실제 IP 주소로 수정

### 3. 배포 실행
```bash
# 의존성 설치
pip install -r requirements.txt

# 연결 테스트
ansible -i inventory/openstack-ha/inventory.ini all -m ping

# 클러스터 배포
ansible-playbook -i inventory/openstack-ha/inventory.ini cluster.yml
```

## 주요 설정

### HA 구성
- **VIP**: 10.0.0.100
- **로드밸런서**: HAProxy
- **API Server 포트**: 6443

### Prometheus 메트릭 엔드포인트
- **API Server**: `https://<master_ip>:6443/metrics`
- **etcd**: `http://<master_ip>:2379/metrics`  
- **Controller Manager**: `https://<master_ip>:10257/metrics`
- **Scheduler**: `https://<master_ip>:10259/metrics`
- **Kubelet**: `https://<node_ip>:10250/metrics`
- **Kube-proxy**: `http://<node_ip>:10249/metrics`

## 문서

- **[배포 가이드](inventory/openstack-ha/DEPLOYMENT_GUIDE.md)**: 상세한 설치 및 설정 가이드

## 지원

이 설정은 다음 환경에서 테스트되었습니다:
- **OS**: Ubuntu 22.04 LTS
- **Kubernetes**: 1.30.8
- **Kubespray**: v2.28.0

---

**참고**: 이 설정은 Kubespray 공식 프로젝트를 기반으로 한 사용자 정의 구성입니다.  
원본 프로젝트: https://github.com/kubernetes-sigs/kubespray