# OpenStack HA Kubernetes Cluster 배포 가이드

## 개요

이 가이드는 OpenStack 환경에서 Kubespray를 사용하여 고가용성(HA) Kubernetes 클러스터를 배포하는 방법을 설명합니다.

## 클러스터 구성

### 인프라 구성
- **총 7대 VM**: bastion(1) + master(3) + worker(3)
- **OS**: Ubuntu 22.04 LTS
- **Kubernetes 버전**: 1.30.8
- **CNI**: Cilium (Helm으로 설치)
- **etcd**: 외부 데몬 방식 3중화

### 네트워크 구성
- **Master VIP**: 10.0.0.100 (PCS로 관리)
- **API Server Port**: 6443
- **LoadBalancer**: HAProxy

## 배포 전 준비사항

### 1. OpenStack VM 생성
```bash
# 7대 VM 생성 (IP는 예시)
# bastion1: 10.0.0.10
# master1:  10.0.0.11
# master2:  10.0.0.12  
# master3:  10.0.0.13
# worker1:  10.0.0.14
# worker2:  10.0.0.15
# worker3:  10.0.0.16
```

### 2. SSH 키 설정
```bash
# Bastion 서버를 통한 SSH 접근 설정
ssh-copy-id ubuntu@10.0.0.10  # bastion
```

### 3. 인벤토리 파일 수정
`inventory/openstack-ha/inventory.ini` 파일에서 실제 IP 주소로 수정:
```ini
[bastion]
bastion1 ansible_host=<BASTION_IP>

[kube_control_plane]
master1 ansible_host=<MASTER1_IP> ip=<MASTER1_IP> etcd_member_name=etcd1
master2 ansible_host=<MASTER2_IP> ip=<MASTER2_IP> etcd_member_name=etcd2  
master3 ansible_host=<MASTER3_IP> ip=<MASTER3_IP> etcd_member_name=etcd3

[kube_node]
worker1 ansible_host=<WORKER1_IP> ip=<WORKER1_IP>
worker2 ansible_host=<WORKER2_IP> ip=<WORKER2_IP>
worker3 ansible_host=<WORKER3_IP> ip=<WORKER3_IP>
```

## 핵심 설정 변경 사항

### 1. HA 로드밸런서 설정 (`group_vars/all/all.yml`)
```yaml
loadbalancer_apiserver:
  address: 10.0.0.100  # VIP 주소
  port: 6443

loadbalancer_apiserver_localhost: false
loadbalancer_apiserver_type: haproxy
```

### 2. Kubernetes 설정 (`group_vars/k8s_cluster/k8s-cluster.yml`)
```yaml
# Kubernetes 버전
kube_version: v1.30.8

# CNI 플러그인
kube_network_plugin: cilium

# Prometheus 메트릭 수집 활성화
kube_controller_manager_metrics_bind_address: "0.0.0.0:10257"
kube_scheduler_metrics_bind_address: "0.0.0.0:10259"
kube_proxy_metrics_bind_address: "0.0.0.0:10249"
kubelet_metrics_bind_address: "0.0.0.0:10250"
```

### 3. etcd 외부 클러스터 설정 (`group_vars/all/etcd.yml`)
```yaml
# 데몬 방식으로 배포 (static pod 아님)
etcd_deployment_type: host

# HA 클러스터 설정
etcd_listen_client_urls: "http://0.0.0.0:2379,https://0.0.0.0:2379"
etcd_listen_peer_urls: "https://0.0.0.0:2380"

# Prometheus 메트릭 활성화
etcd_metrics: basic
```

### 4. Cilium CNI 설정 (`group_vars/k8s_cluster/k8s-net-cilium.yml`)
```yaml
# Helm으로 설치
cilium_deploy_method: helm

# Prometheus 메트릭 활성화
cilium_enable_prometheus: true
```

### 5. 애드온 설정 (`group_vars/k8s_cluster/addons.yml`)
```yaml
# Helm 활성화 (Cilium 설치용)
helm_enabled: true

# 메트릭 서버 활성화
metrics_server_enabled: true
```

## 배포 실행

### 1. 의존성 설치
```bash
pip install -r requirements.txt
```

### 2. 인벤토리 검증
```bash
ansible-inventory -i inventory/openstack-ha/inventory.ini --list
```

### 3. 연결 테스트
```bash
ansible -i inventory/openstack-ha/inventory.ini all -m ping
```

### 4. 클러스터 배포
```bash
ansible-playbook -i inventory/openstack-ha/inventory.ini cluster.yml
```

## 배포 후 검증

### 1. 클러스터 상태 확인
```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

### 2. etcd 클러스터 상태 확인
```bash
# 각 master 노드에서 실행
sudo systemctl status etcd
etcdctl endpoint status --cluster
```

### 3. HA 구성 확인
```bash
# VIP 상태 확인
kubectl cluster-info
```

### 4. CNI 확인
```bash
kubectl get pods -n kube-system | grep cilium
```

### 5. 메트릭 수집 확인
```bash
# API Server 메트릭
curl -k https://10.0.0.100:6443/metrics

# etcd 메트릭
curl http://10.0.0.11:2379/metrics

# Cilium 메트릭
kubectl get svc -n kube-system | grep cilium
```

## 주요 메트릭 엔드포인트

- **API Server**: `https://<master_ip>:6443/metrics`
- **etcd**: `http://<master_ip>:2379/metrics`
- **Controller Manager**: `https://<master_ip>:10257/metrics`
- **Scheduler**: `https://<master_ip>:10259/metrics`
- **Kubelet**: `https://<node_ip>:10250/metrics`
- **Kube-proxy**: `http://<node_ip>:10249/metrics`
- **Cilium**: Service를 통해 노출됨

## 트러블슈팅

### 1. VIP 접근 불가
```bash
# HAProxy 상태 확인
sudo systemctl status haproxy
sudo journalctl -u haproxy -f
```

### 2. etcd 클러스터 문제
```bash
# etcd 로그 확인
sudo journalctl -u etcd -f

# 클러스터 멤버 확인
etcdctl member list
```

### 3. Cilium 네트워크 문제
```bash
# Cilium 상태 확인
kubectl exec -n kube-system cilium-xxxxx -- cilium status
```

## PCS 설정 (추가 작업 필요)

현재 설정은 HAProxy 기반이며, PCS(Pacemaker) 설정은 별도로 구성해야 합니다:

1. **Pacemaker/Corosync 설치**
2. **VIP 리소스 설정**
3. **HAProxy 리소스 설정**

자세한 PCS 설정은 별도 가이드가 필요합니다.

## 추가 고려사항

1. **보안**: 방화벽 규칙 설정
2. **백업**: etcd 정기 백업 설정
3. **모니터링**: Prometheus/Grafana 설치
4. **로깅**: EFK 스택 설치
5. **스토리지**: 퍼시스턴트 볼륨 설정