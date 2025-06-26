# CLAUDE_KR.md

이 파일은 Claude Code (claude.ai/code)가 이 저장소에서 작업할 때 참고할 수 있는 한글 가이드입니다.

## 프로젝트 개요

Kubespray는 프로덕션 환경에 적합한 Kubernetes 클러스터를 배포하기 위한 Ansible 기반 도구입니다. 여러 클라우드 프로바이더와 베어메탈 환경에서 Ansible 플레이북을 사용하여 Kubernetes 구성 요소를 설치하고 구성합니다.

## 핵심 명령어

### 클러스터 관리
- `ansible-playbook -i inventory/sample/inventory.ini cluster.yml` - 새로운 Kubernetes 클러스터 배포
- `ansible-playbook -i inventory/sample/inventory.ini scale.yml` - 클러스터 확장 (노드 추가/제거)
- `ansible-playbook -i inventory/sample/inventory.ini upgrade-cluster.yml` - 클러스터 컴포넌트 업그레이드
- `ansible-playbook -i inventory/sample/inventory.ini reset.yml` - 클러스터 초기화 및 정리
- `ansible-playbook -i inventory/sample/inventory.ini remove-node.yml -e node=nodename` - 특정 노드 제거
- `ansible-playbook -i inventory/sample/inventory.ini recover-control-plane.yml` - 컨트롤 플레인 복구

### 개발 및 테스트
- `cd tests && make create-vagrant` - Vagrant 테스트 환경 생성
- `cd tests && make delete-vagrant` - Vagrant 테스트 환경 삭제
- `molecule test` - Molecule 테스트 실행 (개별 역할 디렉토리에서)
- `ansible-playbook tests/cloud_playbooks/create-kubevirt.yml` - 클라우드 테스트 환경 생성

### 의존성 설치
- Python 요구사항 설치: `pip install -r requirements.txt`
- 테스트 요구사항 설치: `pip install -r tests/requirements.txt`

## 아키텍처

### 주요 플레이북 구조
- `cluster.yml` - 전체 클러스터 배포를 조율하는 메인 진입점
- `playbooks/cluster.yml` - 다음 단계들로 구성된 핵심 배포 플레이북:
  1. **준비** - OS 부트스트랩, 컨테이너 엔진 설정, 다운로드
  2. **etcd 설치** - Kubernetes 상태 저장을 위한 etcd 클러스터 배포
  3. **Kubernetes 설치** - 컨트롤 플레인 및 워커 노드 배포
  4. **CNI 설치** - 컨테이너 네트워크 인터페이스 설치
  5. **애플리케이션** - 클러스터 애플리케이션 및 애드온 배포

### 주요 역할 카테고리
- **bootstrap_os/** - OS별 부트스트래핑 (Ubuntu, CentOS 등)
- **container-engine/** - 컨테이너 런타임 설정 (containerd, Docker, CRI-O)
- **kubernetes/** - 핵심 Kubernetes 구성 요소 (control-plane, node, kubeadm)
- **network_plugin/** - CNI 구현체 (Calico, Cilium, Flannel 등)
- **kubernetes-apps/** - 클러스터 애플리케이션 (DNS, ingress, storage, monitoring)
- **etcd/** - etcd 클러스터 관리 및 구성

### 인벤토리 및 구성
- `inventory/sample/` - 템플릿 인벤토리 및 그룹 변수
- `inventory/sample/group_vars/all/` - 전역 구성 변수
- `inventory/sample/group_vars/k8s_cluster/` - Kubernetes 전용 설정
- 네트워크 플러그인 구성: 다양한 CNI 옵션을 위한 `k8s-net-*.yml` 파일들

### 지원 플랫폼
- **컨테이너 런타임**: containerd (기본값), Docker, CRI-O
- **네트워크 플러그인**: Calico (기본값), Cilium, Flannel, Kube-OVN, Kube-Router, Macvlan
- **클라우드 프로바이더**: AWS, GCP, Azure, OpenStack, vSphere, Hetzner, OCI
- **운영체제**: Ubuntu, Debian, CentOS/RHEL, Fedora, openSUSE, Flatcar

## 개발 워크플로우

1. **구성**: `inventory/sample`을 복사하고 변수를 사용자 정의하는 것부터 시작
2. **테스트**: 배포 테스트를 위해 Vagrant 또는 클라우드 프로바이더 사용
3. **역할**: 개별 구성 요소는 Molecule 테스트와 함께 Ansible 역할로 구성
4. **변수**: 구성은 변수 중심적 - 옵션은 `group_vars/`에서 확인
5. **검증**: `validate_inventory` 역할이 배포 전 구성을 검사

## 중요 사항

- 이것은 전통적인 소프트웨어 애플리케이션이 아닌 Ansible 기반 Infrastructure-as-Code 프로젝트입니다
- 대부분의 "테스트"는 단위 테스트보다는 실제 인프라 배포를 포함합니다
- 구성은 Ansible 인벤토리 파일과 그룹 변수를 통해 관리됩니다
- 이 프로젝트는 변수 사용자 정의를 통해 다양한 배포 시나리오를 지원합니다
- 각 역할은 Molecule 프레임워크를 사용하여 독립적으로 테스트할 수 있습니다