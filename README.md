# 설치

# Ansible?

⇒ opensource 자동화 플랫폼

플레이북은 사람이 쉽게 읽고 변경할 수 있는 자동화 툴

---

## Terraform vs Ansible

| 항목 | 테라폼 | 앤서블 |
| --- | --- | --- |
| 개발 회사 | HashiCorp | Red Hat |
| 주요 사용 용도 | 클라우드 인프라 관리 | 서버 구성 관리, 배포 |
| 도구 종류 | IaC 도구 | IT 자동화 도구 |
| 작업 정의 언어 | HCL | YAML |
| 에이전트 필요성 | 없음 | 없음 |
| 모듈/플러그인 | 프로바이더 | 모듈 |
| 클라우드 지원 | 다양한 클라우드 프로바이더 | 일부 클라우드 프로바이더 |

---

<img width="526" height="334" alt="image" src="https://github.com/user-attachments/assets/2cf631c1-547e-4b6a-80df-5813a734bdda" />

## host.ini(必)

### ⇒ Inventory

관리 대상 서버들을 나열하는 파일

SSH로 접속하기 때문에 미리 키 등록을 해주어야 함

## ansible.cfg

### ⇒ Configuration

동작 방식 규칙을 정함

ex) 인벤토리 경로

---

### 디렉토리 구조

```jsx
ansible-project/
├── ansible.cfg         # 1. Ansible 환경 설정 파일
├── hosts.ini           # 2. 관리 대상 서버 목록 (Inventory)
├── site.yml            # 3. 메인 플레이북 (전체 실행용)
├── webservers.yml      #    특정 그룹용 플레이북 (웹 서버 설정)
├── dbservers.yml       #    특정 그룹용 플레이북 (DB 서버 설정)
├── roles/              # 4. 재사용 가능한 작업 단위 (Roles)
│   └── common/         #    공통 설정 (로그 설정, 사용자 추가 등)
│   └── web/            #    웹 설정 (Nginx 설치 등)
├── group_vars/         # 5. 그룹별 변수 저장 폴더
│   ├── all.yml         #    모든 서버 공통 변수
│   └── webservers.yml  #    webservers 그룹 전용 변수
└── host_vars/          # 6. 개별 호스트별 변수 저장 폴더
    └── 192.168.1.10.yml
```

## roles

⇒ 적을 수 있는 폴더 명들이 정해져 있음

| **폴더명** | **역할 (앤서블의 약속)** | **핵심 파일** |
| --- | --- | --- |
| **`tasks`** | 실제 실행할 명령(Task)이 들어있는 곳 (필수) | `main.yml` |
| **`handlers`** | 서비스 재시작(restart) 같은 **후속 작업** 모음 | `main.yml` |
| **`vars`** | 해당 역할에서 사용할 **중요 변수** | `main.yml` |
| **`defaults`** | 우선순위가 낮은 **기본 변수값** (사용자가 덮어쓰기 가능) | `main.yml` |
| **`files`** | 수정 없이 그대로 서버에 보낼 **일반 파일** (스크립트, 인증서 등) | (제한 없음) |
| **`templates`** | 변수를 치환해서 서버에 보낼 **설정 파일 템플릿** (`.j2`) | `*.j2` |
| **`meta`** | 이 역할의 정보나 **의존성**(다른 역할 먼저 실행 등) 정의 | `main.yml` |
| **`library`** | 직접 만든 커스텀 모듈 (고급 사용자용) | `*.py` |

```jsx
[root@p-jhjeon-master ansible]# tree
.
├── ansible.cfg
├── hosts
├── k8s_prepare.yaml
└── roles
    └── k8s_prepare
        └── **tasks**
            └── main.yaml
```

---

## Ansible로 k8s 기본 세팅하기

```jsx
[root@p-jhjeon-master ansible]# vi hosts

```
[k8s_master]
10.60.100.93

[k8s_worker]
10.60.100.94
10.60.100.95

[k8s_cluster:children]
k8s_master
k8s_worker
```
```

### 폴더 구조 만들기

```jsx
[root@p-jhjeon-master ansible]# tree
.
├── ansible.cfg
├── hosts
├── k8s_prepare.yaml
└── roles
    └── k8s_prepare
        └── tasks
            └── main.yaml

```

**k8s_prepare.yaml: 실행 파일**

**/etc/ansible/roles/k8s_prepare/tasks: 실제 동작 되는 파일들**

### vi hosts

```jsx
[root@p-jhjeon-master ansible]# vi hosts
[k8s_master]
10.60.100.93

[k8s_worker]
10.60.100.94
10.60.100.95

[k8s_cluster:children]
k8s_master
k8s_worker

[all:vars]
ansible_user=root
ansible_password=accordion!@#
ansible_ssh_common_args='-o StrictHostKeyChecking=no'

```

### vi roles/k8s_prepare/tasks/main.yaml

```jsx
- name: Chrony 설치
  dnf:
    name: chrony
    state: present

- name: Chrony 서비스 시작
  service:
    name: chronyd
    state: started
    enabled: yes

- name: 시간 동기화 즉시 적용
  command: chronyc -a makestep

- name: Firewalld 방화벽 비활성화 (Rocky Linux)
  service:
    name: firewalld
    state: stopped
    enabled: no

- name: 현재 실행 중인 Swap 끄기
  command: swapoff -a
  when: ansible_swaptotal_mb > 0

- name: 재부팅 시에도 Swap이 켜지지 않도록 fstab에서 제거
  replace:
    path: /etc/fstab
    regexp: '^([^#].*?\sswap\s+sw\s+.*)$'
    replace: '# \1'

- name: /etc/hosts 도메인 작성
  blockinfile:
    path: /etc/hosts
    block: |
      {% for host in groups['all'] %}
      {{ hostvars[host]['ansible_host'] }} {{ host }}
      {% endfor %}
    
- name: 오버레이 및 브릿지 넷필터 모듈 로드
  modprobe:
    name: "{{ item }}"
    state: present
  loop:
    - overlay
    - br_netfilter

- name: k8s용 sysctl 설정 파일 생성
  copy:
    dest: /etc/sysctl.d/k8s.conf
    content: |
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1

- name: sysctl 설정 적용
  command: sysctl --system

```

### vi k8s_prepare.yaml

```jsx
- name: 쿠버네티스 기본 환경
  hosts: all               
  become: yes
  gather_facts: yes
  roles:
    - k8s_prepare
```

hosts: all               # hosts 파일에 정의된 모든 서버 대상
become: yes             # sudo 권한 사용
gather_facts: yes       # 서버 정보(메모리, OS 등) 수집 활성화

roles:
- k8s_prep             # roles의 경로

### 문법 check

```jsx
[root@p-jhjeon-master ansible]# ansible-playbook k8s_prepare.yaml --syntax-check
[WARNING]: Collection community.general does not support Ansible version 2.14.18

playbook: k8s_prepare.yaml
```

### ansible-playbook k8s_prepare.yaml --check

```jsx
[root@p-jhjeon-master ansible]# ansible-playbook k8s_prepare.yaml --check
[WARNING]: Collection community.general does not support Ansible version 2.14.18

PLAY [쿠버네티스 기본 환경] **********************************************************************************************************************************

TASK [Gathering Facts] ***************************************************************************************************************************************
ok: [10.60.100.93]
ok: [10.60.100.94]
ok: [10.60.100.95]

TASK [k8s_prepare : Chrony 설치] *****************************************************************************************************************************
ok: [10.60.100.94]
ok: [10.60.100.95]
ok: [10.60.100.93]

TASK [k8s_prepare : Chrony 서비스 시작] **********************************************************************************************************************
ok: [10.60.100.95]
ok: [10.60.100.94]
ok: [10.60.100.93]

TASK [k8s_prepare : 시간 동기화 즉시 적용] *******************************************************************************************************************
skipping: [10.60.100.95]
skipping: [10.60.100.94]
skipping: [10.60.100.93]

TASK [k8s_prepare : Firewalld 방화벽 비활성화 (Rocky Linux)] *************************************************************************************************
ok: [10.60.100.94]
ok: [10.60.100.95]
ok: [10.60.100.93]

TASK [k8s_prepare : 현재 실행 중인 Swap 끄기] ****************************************************************************************************************
skipping: [10.60.100.93]
skipping: [10.60.100.94]
skipping: [10.60.100.95]

TASK [k8s_prepare : 재부팅 시에도 Swap이 켜지지 않도록 fstab에서 제거] ***************************************************************************************
ok: [10.60.100.95]
ok: [10.60.100.94]
ok: [10.60.100.93]

TASK [k8s_prepare : 오버레이 및 브릿지 넷필터 모듈 로드] *****************************************************************************************************
ok: [10.60.100.95] => (item=overlay)
ok: [10.60.100.94] => (item=overlay)
ok: [10.60.100.93] => (item=overlay)
ok: [10.60.100.95] => (item=br_netfilter)
changed: [10.60.100.94] => (item=br_netfilter)
ok: [10.60.100.93] => (item=br_netfilter)

TASK [k8s_prepare : k8s용 sysctl 설정 파일 생성] *************************************************************************************************************
changed: [10.60.100.94]
changed: [10.60.100.95]
changed: [10.60.100.93]

TASK [k8s_prepare : sysctl 설정 적용] ************************************************************************************************************************
skipping: [10.60.100.95]
skipping: [10.60.100.94]
skipping: [10.60.100.93]

PLAY RECAP ***************************************************************************************************************************************************
10.60.100.93               : ok=7    changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
10.60.100.94               : ok=7    changed=2    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0
10.60.100.95               : ok=7    changed=1    unreachable=0    failed=0    skipped=3    rescued=0    ignored=0

```

⇒ 실제 반영하지는 않고 되는지 체크할 수 있음

---

## containerd 설치

⇒ https://docs.docker.com/engine/install/rhel/

containerd 설치 시에는 기본 dnf 로는 불가능하기 때문에 docker 저장소 경로를 추가해야함

```jsx
[root@p-jhjeon-master ansible]# vi roles/containerd/tasks/main.yaml
```
- name: "dnf-plugins-core 설치"
  dnf:
    name: dnf-plugins-core
    state: present

- name: "Docker 저장소 추가"
  shell: dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
  args:
    creates: /etc/yum.repos.d/docker-ce.repo

- name: containerd 설치
  dnf:
    name: containerd.io
    state: present

- name: config.toml 설정
  shell: "containerd config default > /etc/containerd/config.toml"
  args:
    creates: /etc/containerd/config.toml

- name: SystemdCgroup  true
  replace:
    path: /etc/containerd/config.toml
    regexp: 'SystemdCgroup = false'
    replace: 'SystemdCgroup = true'

- name: containerd 서비스 시작
  service:
    name: containerd
    state: started
    enabled: yes

```
```

```jsx
 [root@p-jhjeon-master ansible]# vi containerd.yaml
```
 name: install containerd
  hosts: k8s_cluster
  become: yes
  roles:
    - containerd
```
```

---

## k8s 설치

### https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

```jsx
[root@p-jhjeon-master ansible]# tree
.
├── ansible.cfg
├── containerd.yaml
├── hosts
├── k8s_install.yaml
├── k8s_prepare.yaml
└── roles
    ├── containerd
    │   └── tasks
    │       └── main.yaml
    ├── k8s_install
    │   └── tasks
    │       └── main.yaml
    └── k8s_prepare
        └── tasks
            └── main.yaml

```

```jsx
[root@p-jhjeon-master ansible]# vi k8s_install.yaml
- name: install containerd
  hosts: k8s_cluster
  become: yes
  roles:
    - k8s_install
```

```jsx
[root@p-jhjeon-master ansible]# vi roles/k8s_install/tasks/main.yaml
```
- name: SELinux Permissive
  shell: setenforce 0
  ignore_errors: yes

- name: SELinux 설정 영구 적용
  replace:
    path: /etc/selinux/config
    regexp: '^SELINUX=enforcing$'
    replace: 'SELINUX=permissive'

- name: Kubernetes 1.35
  yum_repository:
    name: kubernetes
    description: Kubernetes
    baseurl: https://pkgs.k8s.io/core:/stable:/v1.35/rpm/
    enabled: yes
    gpgcheck: yes
    gpgkey: https://pkgs.k8s.io/core:/stable:/v1.35/rpm/repodata/repomd.xml.key
    exclude: kubelet kubeadm kubectl cri-tools kubernetes-cni

- name: kubelet, kubeadm, kubectl
  dnf:
    name:
      - kubelet
      - kubeadm
      - kubectl
    state: present
    disable_excludes: kubernetes

- name: kubelet 활성화
  service:
    name: kubelet
    state: started
    enabled: yes
```
```

```jsx
[root@p-jhjeon-master ansible]# ansible-playbook k8s_install.yaml --check

PLAY [install containerd] **************************************************************************************************

TASK [Gathering Facts] *****************************************************************************************************
ok: [10.60.100.94]
ok: [10.60.100.95]
ok: [10.60.100.93]

TASK [k8s_install : SELinux Permissive] ************************************************************************************
skipping: [10.60.100.94]
skipping: [10.60.100.95]
skipping: [10.60.100.93]

TASK [k8s_install : SELinux 설정 영구 적용] ********************************************************************************
ok: [10.60.100.95]
ok: [10.60.100.94]
ok: [10.60.100.93]

TASK [k8s_install : Kubernetes 1.35] ***************************************************************************************
ok: [10.60.100.95]
ok: [10.60.100.94]
changed: [10.60.100.93]

TASK [k8s_install : kubelet, kubeadm, kubectl] *****************************************************************************
ok: [10.60.100.95]
ok: [10.60.100.94]
ok: [10.60.100.93]

TASK [k8s_install : kubelet 활성화] ****************************************************************************************
ok: [10.60.100.95]
ok: [10.60.100.94]
ok: [10.60.100.93]

PLAY RECAP *****************************************************************************************************************
10.60.100.93               : ok=5    changed=1    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
10.60.100.94               : ok=5    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
10.60.100.95               : ok=5    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0

[root@p-jhjeon-master ansible]# ansible-playbook k8s_install.yaml --syntax-check

playbook: k8s_install.yaml

```

### kubeadm join

```jsx
[root@p-jhjeon-master ansible]# vi roles/k8s_master/tasks/main.yaml
```
- name: kubeadm 초기화
  shell: |
    kubeadm init --pod-network-cidr=192.168.0.0/16 \
                 --apiserver-advertise-address=10.60.100.93
  register: init_result
  args:
    creates: /etc/kubernetes/admin.conf

- name: kubectl 설정 폴더
  file:
    path: "{{ ansible_env.HOME }}/.kube"
    state: directory

- name: 관리자 설정 파일 복사
  copy:
    src: /etc/kubernetes/admin.conf
    dest: "{{ ansible_env.HOME }}/.kube/config"
    remote_src: yes

- name: 조인 명령 실행
  shell: kubeadm token create --print-join-command
  register: join_command_raw

- name: 조인 명령어를 메모리에 저장
  set_fact:
    k8s_join_command: "{{ join_command_raw.stdout }}"
```
```

```jsx
[root@p-jhjeon-master ansible]# vi roles/k8s_worker/tasks/main.yaml
---
- name:  클러스터에 조인
  shell: "{{ hostvars['master-node']['k8s_join_command'] }}"
  args:
    creates: /etc/kubernetes/kubelet.con
```

```jsx
[root@p-jhjeon-master ansible]# tree
.
├── ansible.cfg
├── containerd.yaml
├── hosts
├── k8s_install.yaml
├── k8s_prepare.yaml
├── roles
│   ├── containerd
│   │   └── tasks
│   │       └── main.yaml
│   ├── k8s_install
│   │   └── tasks
│   │       └── main.yaml
│   ├── k8s_master
│   │   └── tasks
│   │       └── main.yaml
│   ├── k8s_prepare
│   │   └── tasks
│   │       └── main.yaml
│   └── k8s_worker
│       └── tasks
│           └── main.yaml
└── site.yaml

- name: kubeadm 초기화
  shell: |
    kubeadm init --pod-network-cidr=192.168.0.0/16 \
                 --apiserver-advertise-address=10.60.100.93
  register: init_result
  args:
    creates: /etc/kubernetes/admin.conf

- name: kubectl 설정 폴더
  file:
    path: "{{ ansible_env.HOME }}/.kube"
    state: directory

- name: 관리자 설정 파일 복사
  copy:
    src: /etc/kubernetes/admin.conf
    dest: "{{ ansible_env.HOME }}/.kube/config"
    remote_src: yes

- name: 조인 명령 실행
  shell: kubeadm token create --print-join-command
  register: join_command_raw

- name: 조인 명령어를 메모리에 저장
  set_fact:
    k8s_join_command: "{{ join_command_raw.stdout }}"

```

### k8s_join.yaml

```jsx
[root@p-jhjeon-master ansible]# vi k8s_join.yaml
```
# 1. 마스터에서 토큰 생성 및 변수 저장
- name: "마스터 노드 초기화 및 토큰 추출"
  hosts: k8s_master
  become: yes
  roles:
    - k8s_master

# 2. 워커에서 마스터의 토큰을 받아 조인 실행
- name: "워커 노드 클러스터 조인"
  hosts: k8s_worker
  become: yes
  roles:
    - k8s_worker
```
```

## ansible.cfg

```jsx
[defaults]
inventory = ./hosts

host_key_checking = False
```

inventory = ./hosts ⇒ 인벤토리 파일경로

host_key_checking = False ⇒ ssh 로 접속시 yes 입력 생략

## Master & Worker 분리하기

⇒ Master와 Worker에서 실행 되어야하는 명령어가 다르기 때문임

- master

```jsx
[root@jihun-mst01 Ansible-k8s]# vi roles/k8s_master/tasks/main.yaml
---
- name: kubeadm 초기화
  shell: |
    kubeadm init
  register: init_result
  args:
    creates: /etc/kubernetes/admin.conf

- name: kubectl 설정 폴더
  file:
    path: "{{ ansible_env.HOME }}/.kube"
    state: directory

- name: 관리자 설정 파일 복사
  copy:
    src: /etc/kubernetes/admin.conf
    dest: "{{ ansible_env.HOME }}/.kube/config"
    remote_src: yes

- name: 조인 명령 실행
  shell: kubeadm token create --print-join-command
  register: join_command_raw

- name: 조인 명령어를 메모리에 저장
  set_fact:
    k8s_join_command: "{{ join_command_raw.stdout }}"

- name: alias 등록
  blockinfile:
    path: ~/.bashrc
    marker: "# {mark} ANSIBLE MANAGED KUBECTL ALIAS"
    block: |
      # kubectl 단축어 설정
      alias k='kubectl'

      # kubectl 자동 완성(Tab) 설정
      source <(kubectl completion bash)

      # 단축어 'k'에서도 자동 완성이 되도록 설정
      complete -o default -F __start_kubectl k

```

- worker

```jsx
[root@jihun-mst01 Ansible-k8s]# vi roles/k8s_worker/tasks/main.yaml
---
- name:  클러스터에 조인
  shell: "{{ hostvars['master-node']['k8s_join_command'] }}"
  args:
    creates: /etc/kubernetes/kubelet.conf
```

## calico 설정

```jsx
[root@jihun-mst01 Ansible-k8s]# tree
.
├── ansible.cfg
├── containerd.yaml
├── hosts
├── k8s_install.yaml
├── k8s_join.yaml
├── k8s_prepare.yaml
├── roles
│   ├── containerd
│   │   └── tasks
│   │       └── main.yaml
│   ├── k8s_calico
│   │   └── tasks
│   │       └── main.yaml
│   ├── k8s_install
│   │   └── tasks
│   │       └── main.yaml
│   ├── k8s_master
│   │   └── tasks
│   │       └── main.yaml
│   ├── k8s_prepare
│   │   └── tasks
│   │       └── main.yaml
│   └── k8s_worker
│       └── tasks
│           └── main.yaml
└── site.yaml

```

```jsx
[root@jihun-mst01 Ansible-k8s]# vi roles/k8s_calico/tasks/main.yaml
- name: calico  설치
  become: false
  command: kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.4/manifests/calico.yaml
```

### cluster.yaml

```jsx
[root@p-jhjeon-master ansible]# vi cluster.yaml
---
# 1. [공통] 모든 노드(마스터, 워커) 환경 준비
- name: "Step 1: 모든 노드 기본 설정 및 엔진 설치"
  hosts: k8s_cluster       # hosts 파일의 모든 서버 그룹
  become: yes              # root 권한 사용
  roles:
    - k8s_prepare          # 방화벽, 스왑, /etc/hosts 설정
    - containerd           # 컨테이너 엔진 설치
    - k8s_install          # kubeadm, kubelet 패키지 설치

# 2. [마스터] 93번 서버 초기화
- name: "Step 2: 마스터 노드 초기화 및 토큰 생성"
  hosts: k8s_master        # hosts 파일의 [k8s_master] 그룹
  become: yes
  roles:
    - k8s_master           # kubeadm init 및 조인 명령어 추출

# 3. [워커] 94, 95번 서버 클러스터 합류
- name: "Step 3: 워커 노드 클러스터 조인"
  hosts: k8s_worker        # hosts 파일의 [k8s_worker] 그룹
  become: yes
  roles:
    - k8s_worker           # 마스터가 만든 토큰으로 조인 실행

```

해당 roles를 어떤 hosts에 적용할지 정할 수 있는 파일

ex) join은 워커에서만 필요하기 때문에 마스터에서 하면 안됨 

kubeinit은 마스터에서만 해야함

## k8s 설치 完

```jsx
[root@jihun-mst01 Ansible-k8s]# kubectl get po -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-58d87d69dc-hbj7w   1/1     Running   0          11m
kube-system   calico-node-mvtq4                          1/1     Running   0          11m
kube-system   calico-node-wgmdd                          1/1     Running   0          11m
kube-system   coredns-7d764666f9-9f8pr                   1/1     Running   0          21m
kube-system   coredns-7d764666f9-jlgjn                   1/1     Running   0          21m
kube-system   etcd-jihun-mst01                           1/1     Running   0          21m
kube-system   kube-apiserver-jihun-mst01                 1/1     Running   0          21m
kube-system   kube-controller-manager-jihun-mst01        1/1     Running   0          21m
kube-system   kube-proxy-2v57p                           1/1     Running   0          21m
kube-system   kube-proxy-bbjfc                           1/1     Running   0          21m
kube-system   kube-scheduler-jihun-mst01                 1/1     Running   0          21m

```

---

## kubectl neat

⇒ k8s에 띄어져있는 야물파일에서 필요한것만 추출할 수 있는 플러그인

그냥 edit이나 -o yaml로 조회하게 되면 불필요한 데이터들이 많이 생김

https://github.com/itaysk/kubectl-neat

```jsx
# 1. 
curl -Lo kubectl-neat.tar.gz https://github.com/itaysk/kubectl-neat/releases/latest/download/kubectl-neat_linux_amd64.tar.gz

# 2. 압축 풀기
tar -xvf kubectl-neat.tar.gz

# 3. 실행 파일을 /usr/local/bin으로 이동 
sudo mv kubectl-neat /usr/local/bin/kubectl-neat

# 4. 설치 확인
kubectl neat --help
```

```jsx
[root@p-jhjeon-master ansible]# for deploy in apache-deploy mariadb-deploy tomcat-deploy; do
  kubectl get deploy $deploy -o yaml | kubectl neat > ./roles/k8s_webwas/templates/${deploy}.yaml.j2
done
[root@p-jhjeon-master ansible]# for svc in apache-svc mariadb-svc tomcat-svc; do
  kubectl get svc $svc -o yaml | kubectl neat > ./roles/k8s_webwas/templates/${svc}.yaml.j2
done
[root@p-jhjeon-master ansible]# for cm in apache-config tomcat-cm tomcat-web-cm; do
  kubectl get cm $cm -o yaml | kubectl neat > ./roles/k8s_webwas/templates/${cm}.yaml.j2
done
```

~> 수동으로 만들었던 webwasdb에서 중요 부분만 야물로 추출해서 ansible로 가져옴

### 배포 파일 생성

```jsx
[root@jihun-mst01 Ansible-k8s]# vi webwas_deploy.yaml
---
- name: "Kubernetes WEB-WAS-DB 배포"
  hosts: k8s_master
  become: yes
  roles:
    - k8s_webwas
```

```jsx
│   ├── k8s_webwas
│   │   ├── files
│   │   │   ├── ROOT.war
│   │   │   └── mariadb-java-client-3.5.7.jar
│   │   ├── tasks
│   │   │   └── main.yaml
│   │   └── templates
│   │       ├── apache-config.yaml.j2
│   │       ├── apache-deploy.yaml.j2
│   │       ├── apache-svc.yaml.j2
│   │       ├── mariadb-deploy.yaml.j2
│   │       ├── mariadb-svc.yaml.j2
│   │       ├── tomcat-cm.yaml.j2
│   │       ├── tomcat-deploy.yaml.j2
│   │       ├── tomcat-svc.yaml.j2
│   │       └── tomcat-web-cm.yaml.j2
```

⇒ tomcat 파일과 db연결에 필요한 드라이버를 미리 앤서블 폴더에 넣어둠

### tasks/main.yaml

```jsx
[root@jihun-mst01 Ansible-k8s]# vi roles/k8s_webwas/tasks/main.yaml

---
- name: "1. 배포용 임시 디렉토리 생성"
  file:
    path: /root/web-was-deploy
    state: directory
# 1. 작업 디렉토리 및 NFS 경로 초기화 (오염 방지)
- name: "기존 NFS 데이터 및 임시 디렉토리 초기화"
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /root/web-was-deploy
    - /root/nfs-share/ROOT
    - /root/nfs-share/lib

- name: "NFS 필수 디렉토리 생성"
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - /root/web-was-deploy
    - /root/nfs-share/ROOT
    - /root/nfs-share/lib

# 2. 파일 전송 (Controller -> Master Node)
- name: "WAR 파일 및 JDBC 드라이버를 NFS 서버로 전송"
  copy:
    src: "{{ item.src }}"
    dest: "{{ item.dest }}"
    mode: '0644'
  loop:
    - { src: "ROOT.war", dest: "/root/nfs-share/ROOT/ROOT.war" }
    - { src: "mariadb-java-client-3.5.7.jar", dest: "/root/nfs-share/lib/mariadb-java-client.jar" }

# 3. WAR 파일 압축 해제 및 드라이버 주입
- name: "WAR 파일 압축 해제"
  unarchive:
    src: /root/nfs-share/ROOT/ROOT.war
    dest: /root/nfs-share/ROOT/
    remote_src: yes

- name: "애플리케이션 라이브러리에 드라이버 복사 (이중 안전)"
  copy:
    src: /root/nfs-share/lib/mariadb-java-client.jar
    dest: /root/nfs-share/ROOT/WEB-INF/lib/mariadb-java-client.jar
    remote_src: yes
    mode: '0644'

# 4. K8s 템플릿 변환 및 배포
- name: "J2 템플릿 변환"
  template:
    src: "{{ item }}"
    dest: "/root/web-was-deploy/{{ item | basename | replace('.j2', '') }}"
  with_fileglob: ["../templates/*.j2"]

- name: "기존 K8s 리소스 삭제 및 재생성 (교체 배포)"
  shell: "kubectl delete -f /root/web-was-deploy/{{ item }} --ignore-not-found=true"
  loop: [ "apache-deploy.yaml", "tomcat-deploy.yaml", "mariadb-deploy.yaml" ]
  failed_when: false

- name: "리소스 정리 대기"
  pause: { seconds: 5 }

- name: "Kubernetes 리소스 적용"
  shell: "kubectl apply -f /root/web-was-deploy/{{ item }}"
  loop:
    - apache-config.yaml
    - tomcat-cm.yaml
    - tomcat-web-cm.yaml
    - mariadb-svc.yaml
    - tomcat-svc.yaml
    - apache-svc.yaml
    - mariadb-deploy.yaml
    - tomcat-deploy.yaml
    - apache-deploy.yaml

```

## 변수 설정

Ansible에서는 group_vars라는 폴더 이름으로 변수를 저장할 수 있음

```jsx
[root@jihun-mst01 Ansible-k8s]# vi group_vars/all.yaml

---
nfs_server_ip: "10.140.20.121"      # 마스터 노드 IP (NFS 서버)

# --- Apache (Web) 설정 ---
apache_node_port: 31450            # 외부에서 접속할 포트 번호
apache_replicas: 1                 # 아파치 서버 대수

# --- Tomcat (WAS) 설정 ---
tomcat_replicas: 1                 # 톰캣 서버 대수
tomcat_version: "9.0"              # 사용할 톰캣 버전

# --- MariaDB (DB) 설정 ---
db_root_password: "pass123#"       # DB 루트 비밀번호
db_name: "test"                    # 초기 생성할 데이터베이스 이름
mariadb_version: "10.6"            # 마리아디비 버전

```

⇒ 아래와 같이 매번 환경이 변경되는 곳에서도 쉽게 적용할 수 있도록 deploy와 svc와 같은 것들은 변수 처리를 함

```jsx
        nfs:
          path: /root/nfs-share
          server: {{ nfs_server_ip }}
```

https://github.com/xinun/Ansible-k8s.git
