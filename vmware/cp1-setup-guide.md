# VMware cp1 서버 생성 및 고정 IP 설정 가이드

이 가이드는 VMware Workstation/Fusion에서 cp1 (Control Plane 1) 서버를 생성하고  
**vmnet1 (Host-Only) + vmnet8 (NAT)** 2개 네트워크를 함께 사용하여 다음과 같이 구성하는 방법을 안내합니다.

- `vmnet1 (Host-Only)`: 고정 IP `192.168.56.10/24`
- `vmnet8 (NAT)`: 인터넷 연결용 DHCP

---

## 1. VMware 네트워크 설정 (vmnet1 + vmnet8)

### Windows/Linux — VMware Workstation Pro

1. **Virtual Network Editor 실행**
   ```
   Edit → Virtual Network Editor
   (또는 관리자 권한으로 vmnetcfg.exe 실행)
   ```

2. **vmnet1 (Host-only), vmnet8 (NAT) 설정 확인/수정**
   ```
   ┌─────────────────────────────────────────────────┐
   │ Network          Type        Subnet Address     │
   │ vmnet0          Bridged     (Auto)              │
   │ vmnet1 ✓        Host-only   192.168.56.0       │  ← 이것 선택
   │ vmnet8          NAT         192.168.146.0       │
   └─────────────────────────────────────────────────┘
   ```

3. **vmnet1 상세 설정**
   - **Subnet IP**: `192.168.56.0`
   - **Subnet mask**: `255.255.255.0`
   - **Host IP**: `192.168.56.1` (호스트 PC의 vmnet1 어댑터 IP)

4. **vmnet8 상세 설정**
   - **Subnet IP**: `192.168.146.0`
   - **Subnet mask**: `255.255.255.0`
   - **Gateway IP**: `192.168.146.2`
   - **DHCP**: Enabled

5. **vmnet1 DHCP 비활성화** (선택사항, 고정 IP 사용 시 권장)
   - "Use local DHCP service to distribute IP address to VMs" 체크 해제

6. **Apply → OK** 클릭

### macOS — VMware Fusion Pro

1. **네트워크 설정 열기**
   ```
   VMware Fusion → Settings → Network
   ```

2. **vmnet1 확인**
   ```
   ┌──────────────────────────────────────────┐
   │ Custom (vmnet1)                          │
   │ Connect the host Mac to a private network│
   │ Subnet IP: 192.168.56.0                  │
   │ Subnet Mask: 255.255.255.0               │
   └──────────────────────────────────────────┘
   ```

3. 필요시 "Show All" → "vmnet1" 편집하여 서브넷 변경

---

## 2. cp1 VM 생성

### 방법 A — GUI로 생성

#### Step 1: 새 VM 생성

```
VMware Workstation:
  File → New Virtual Machine → Custom (Advanced)

VMware Fusion:
  File → New → Create a custom virtual machine
```

#### Step 2: 가상 머신 설정

| 항목 | 설정 값 |
|------|---------|
| **VM 이름** | `cp1` |
| **Guest OS** | Linux → Ubuntu 64-bit |
| **Hardware Compatibility** | Workstation 17.x / Fusion 13.x |
| **메모리** | 4096 MB (4 GB) |
| **프로세서** | 2 cores |
| **네트워크 어댑터** | **Adapter 1: Host-only (vmnet1)** / **Adapter 2: NAT (vmnet8)** |
| **디스크** | 40 GB, Store as a single file |
| **ISO 이미지** | Ubuntu 24.04 Server ISO |

⚠️ **네트워크 어댑터 설정 필수**:
```
VM Settings
  → Network Adapter 1: Custom (vmnet1) 또는 Host-only
  → Add → Network Adapter
  → Network Adapter 2: NAT (vmnet8)
```

#### Step 3: VM 시작 및 Ubuntu 설치

1. VM 전원 켜기
2. Ubuntu 24.04 설치 진행
3. 설치 중 네트워크 설정:
   - **DHCP로 임시 설치** (설치 완료 후 고정 IP로 변경)
   - 또는 설치 중 수동 IP 설정 가능

---

## 3. 고정 IP 설정 (192.168.56.10)

Ubuntu 24.04는 **Netplan**을 사용하여 네트워크를 설정합니다.

### Step 1: VM 접속

```bash
# VM 콘솔에서 로그인하거나
# 임시 DHCP IP로 SSH 접속
ssh ubuntu@<DHCP-IP>
```

### Step 2: 네트워크 인터페이스 이름 확인

```bash
ip addr show

# 출력 예시:
# 1: lo: ...
# 2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
#     inet 192.168.56.10/24 ...
# 3: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
#     inet 192.168.146.xxx/24 ...
```

> 인터페이스 이름은 `ens33`, `ens160`, `enp0s3` 등 시스템에 따라 다를 수 있습니다.

### Step 3: Netplan 설정 파일 생성

#### 방법 A: cat 명령어 사용 (권장 - 복사/붙여넣기 편함)

```bash
sudo cat > /etc/netplan/01-netcfg.yaml <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      addresses:
        - 192.168.56.10/24
      dhcp4: false
    ens160:
      dhcp4: true
      dhcp4-overrides:
        route-metric: 100
EOF
```

> ⚠️ **인터페이스 이름 확인**: `ens33`, `ens160`을 실제 인터페이스 이름으로 변경해야 합니다.

#### 방법 B: nano 에디터 사용

```bash
sudo nano /etc/netplan/01-netcfg.yaml
```

다음 내용 입력:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:                       # Host-only NIC, 실제 이름으로 변경
      addresses:
        - 192.168.56.10/24       # 고정 IP
      dhcp4: false               # DHCP 비활성화
    ens160:                      # NAT NIC, 실제 이름으로 변경
      dhcp4: true
      dhcp4-overrides:
        route-metric: 100
```

> ⚠️ **YAML 주의사항**: 들여쓰기는 **스페이스 2칸**입니다. 탭(Tab) 사용 금지.

### Step 4: 설정 적용

```bash
# 설정 파일 문법 검사
sudo netplan try

# 문법이 올바르면 "Configuration accepted" 메시지 표시
# 120초 이내 Enter 키 입력하여 확정
```

```bash
# 설정 적용
sudo netplan apply
```

### Step 5: IP 확인

```bash
ip addr show ens33
ip addr show ens160

# 출력 예시:
# 2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
#     inet 192.168.56.10/24 brd 192.168.56.255 scope global ens33
# 3: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
#     inet 192.168.146.xxx/24 brd 192.168.146.255 scope global dynamic ens160
```

```bash
# 게이트웨이 확인
ip route

# 출력 예시:
# default via 192.168.146.2 dev ens160 proto dhcp metric 100
# 192.168.56.0/24 dev ens33 proto kernel scope link src 192.168.56.10
```

### Step 6: 연결 테스트

**VM에서 호스트 전용 네트워크 확인**:
```bash
ping -c 4 192.168.56.1
```

**VM에서 인터넷 연결 확인**:
```bash
ping -c 4 8.8.8.8
curl -I https://google.com
```

**호스트에서 VM으로 ping** (호스트 PC 터미널):
```bash
ping -c 4 192.168.56.10
```

**호스트에서 VM으로 SSH 접속**:
```bash
ssh ubuntu@192.168.56.10
```

---

## 4. hostname 변경 (선택)

```bash
# 현재 hostname 확인
hostname

# cp1으로 변경
sudo hostnamectl set-hostname cp1

# /etc/hosts 파일 수정
sudo nano /etc/hosts

# 다음 줄 추가:
127.0.0.1   localhost
127.0.1.1   cp1
192.168.56.10   cp1
```

재부팅 후 확인:
```bash
sudo reboot
# 재접속 후
hostname
# 출력: cp1
```

---

## 5. 고정 NAT 주소가 필요한 경우 (선택)

기본 권장 구성은 `ens160`에서 DHCP를 받는 방식입니다.  
다만 NAT 주소도 고정으로 관리하고 싶다면 아래처럼 설정할 수 있습니다.

```bash
sudo cat > /etc/netplan/01-netcfg.yaml <<'EOF'
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      addresses:
        - 192.168.56.10/24
      dhcp4: false
    ens160:
      addresses:
        - 192.168.146.10/24
      routes:
        - to: default
          via: 192.168.146.2
          metric: 100
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
      dhcp4: false
EOF
```

> ⚠️ **인터페이스 이름**: `ens33`과 `ens160`을 실제 인터페이스 이름으로 변경하세요. (`ip addr show`로 확인)

```bash
sudo netplan apply
```

**확인**:
```bash
# 라우팅 테이블
ip route

# 인터넷 연결
ping -c 4 8.8.8.8
curl -I https://google.com
```

---

## 6. Kubernetes 설치 (다음 단계)

고정 IP 설정이 완료되면 [../vmware/README.md](README.md)의  
**"4. VM 내부 Kubernetes 설치"** 섹션을 참고하여 kubeadm을 설치합니다.

```bash
# containerd, kubeadm, kubelet, kubectl 설치
# (자세한 내용은 README.md 참조)

# Control Plane 초기화 시 --apiserver-advertise-address 명시
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.56.10
```

---

## 7. 문제 해결

### SSH Connection Refused

**증상**: `ssh ubuntu@192.168.56.10` 실행 시 "Connection refused" 오류

#### 진단 1: VM이 실행 중인지 확인

VMware Workstation에서 cp1 VM의 전원 상태 확인
- VM이 꺼져 있으면 → **Power On** 클릭

#### 진단 2: VM 콘솔에서 네트워크 확인

VMware 콘솔에서 VM에 직접 로그인 후:

```bash
# 네트워크 인터페이스 확인
ip addr show

# 192.168.56.10이 할당되어 있는지 확인
# 출력 예시에 "inet 192.168.56.10/24"가 있어야 함
```

IP가 없거나 다른 IP인 경우:
```bash
# Netplan 설정 확인
cat /etc/netplan/01-netcfg.yaml

# 설정 재적용
sudo netplan apply

# 다시 확인
ip addr show
```

#### 진단 3: 호스트에서 VM으로 ping 테스트

Windows PowerShell에서:
```powershell
ping 192.168.56.10
```

- **ping이 안 되면** → 네트워크 설정 문제
  - VM 콘솔에서 Step 3 (Netplan 설정) 다시 확인
  - VMware 네트워크 어댑터가 vmnet1로 설정되어 있는지 확인
  
- **ping은 되는데 SSH가 안 되면** → SSH 서비스 문제 (아래 진단 4)

#### 진단 4: SSH 서비스 확인 (VM 콘솔)

```bash
# SSH 서비스 상태 확인
sudo systemctl status ssh

# SSH 서비스가 없다면 설치
sudo apt update
sudo apt install -y openssh-server

# SSH 서비스 시작 및 활성화
sudo systemctl start ssh
sudo systemctl enable ssh

# SSH 포트 확인 (22번 포트 리스닝 확인)
sudo ss -tlnp | grep :22
# 또는
sudo netstat -tlnp | grep :22
```

#### 진단 5: 방화벽 확인 (VM 콘솔)

```bash
# UFW 방화벽 상태 확인
sudo ufw status

# UFW가 활성화되어 있고 SSH가 차단된 경우
sudo ufw allow 22/tcp
sudo ufw reload

# 또는 UFW 비활성화 (테스트용)
sudo ufw disable
```

#### 진단 6: Windows 호스트 vmnet1 어댑터 확인

Windows PowerShell (관리자 권한):
```powershell
# vmnet1 어댑터 확인
ipconfig /all | Select-String -Pattern "VMnet1" -Context 5,10

# vmnet1 IP가 192.168.56.1인지 확인
# 없으면 Virtual Network Editor에서 재설정
```

### SSH Connection Refused (WSL 환경)

**문제**: WSL (Windows Subsystem for Linux)에서 `ssh ubuntu@192.168.56.10` 실행 시 "Connection refused" 오류 발생

**원인**: WSL2는 자체 가상 네트워크를 사용하므로 Windows 호스트의 VMware 네트워크 어댑터(vmnet1)에 **직접 접근할 수 없습니다**.

**해결 방법**:

#### 방법 1: Windows PowerShell/CMD에서 SSH 사용 (권장)

```powershell
# Windows PowerShell 또는 CMD에서 실행
ssh ubuntu@192.168.56.10
```

Windows 10/11에는 OpenSSH 클라이언트가 기본 설치되어 있습니다.

#### 방법 2: Windows Terminal 설치 (더 나은 터미널 환경)

```powershell
# Microsoft Store에서 "Windows Terminal" 설치
# 또는 winget 사용
winget install Microsoft.WindowsTerminal

# Windows Terminal 실행 후 PowerShell 탭에서
ssh ubuntu@192.168.56.10
```

#### 방법 3: WSL에서 Windows SSH 사용

```bash
# WSL에서 Windows의 SSH.exe 호출
/mnt/c/Windows/System32/OpenSSH/ssh.exe ubuntu@192.168.56.10

# 또는 alias 설정
echo "alias wssh='/mnt/c/Windows/System32/OpenSSH/ssh.exe'" >> ~/.bashrc
source ~/.bashrc
wssh ubuntu@192.168.56.10
```

#### 방법 4: VM에 NAT 어댑터 추가 (WSL에서 접속 필요 시)

기본 구성에 이미 NAT 어댑터가 있다면, VMware에서 포트포워딩만 추가하면 됩니다.

```
Edit → Virtual Network Editor → vmnet8 (NAT) → NAT Settings
  → Port Forwarding → Add
     Host Port: 2222
     VM IP: 192.168.146.10   # 또는 ens160의 DHCP로 받은 NAT IP
     VM Port: 22
```

WSL에서 접속:
```bash
# Windows 호스트의 IP로 접속
ssh -p 2222 ubuntu@$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')

# 또는 간단히
ssh -p 2222 ubuntu@localhost  # WSL2의 경우 작동할 수 있음
```

### vmnet1 어댑터가 보이지 않음 (Windows)

```bash
# 관리자 권한 PowerShell
Get-NetAdapter | Where-Object {$_.InterfaceDescription -like "*VMware*"}

# vmnet1 재시작
Restart-NetAdapter -Name "VMware Network Adapter VMnet1"
```

### Netplan 적용 후 네트워크 끊김

```bash
# VM 콘솔에서 접속 (SSH 안 될 경우)
# 설정 롤백
sudo netplan --debug apply
sudo rm /etc/netplan/01-netcfg.yaml
sudo netplan generate
sudo netplan apply
```

### 게이트웨이로 ping 안 됨

호스트 PC의 방화벽 확인:
- **Windows**: Windows Defender Firewall → "VMware Authorization Service" 허용
- **Linux**: `ufw allow from 192.168.56.0/24`

### DHCP를 완전히 제거하고 싶음

```bash
# 기존 DHCP 설정 파일 삭제
sudo rm /etc/netplan/00-installer-config.yaml
sudo rm /etc/netplan/50-cloud-init.yaml

# Netplan 재생성
sudo netplan generate
sudo netplan apply
```

---

## 요약

| 항목 | 값 |
|------|-----|
| VM 이름 | `cp1` |
| 네트워크 | `vmnet1 (Host-only)` + `vmnet8 (NAT)` |
| Host-only IP | `192.168.56.10/24` |
| NAT IP | DHCP 또는 `192.168.146.10/24` |
| 기본 게이트웨이 | `192.168.146.2` (NAT) |
| 인터페이스 | `ens33`, `ens160` (시스템에 따라 다름) |
| Netplan 파일 | `/etc/netplan/01-netcfg.yaml` |

**접속 방법** (호스트 PC에서):
```bash
ssh ubuntu@192.168.56.10
```

---

## 참고 자료

- [VMware Virtual Networking Concepts](https://docs.vmware.com/en/VMware-Workstation-Pro/17/com.vmware.ws.using.doc/GUID-D9B0A52D-38A2-45D7-A9EB-987ACE77F93C.html)
- [Ubuntu Netplan Documentation](https://netplan.readthedocs.io/)
- [Kubernetes kubeadm Installation](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
