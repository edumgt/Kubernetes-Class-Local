3) 제일 확실한 방법(추천): SSH로 접속해서 Windows 터미널에서 복붙

서버용 Ubuntu에서는 이게 “정답 루트”입니다.

3-1) cp1에 SSH 설치

cp1에서:

sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh

3-2) Windows에서 접속

PowerShell에서:

ssh ubuntu@192.168.56.10


(아직 IP를 192.168.56.10으로 안 바꿨으면 현재 IP인 192.168.56.101로 접속)

Windows 터미널/PowerShell에서는 Ctrl+C / Ctrl+V가 그대로 복붙 됩니다. 👍

4) “우분투 GUI(Desktop)”면 Guest Additions로 완벽하게

만약 Desktop을 깔았다면:

Devices → Insert Guest Additions CD image…

게스트 안에서 설치 후 재부팅
하면 양방향 클립보드가 아주 잘 됩니다.

하지만 지금처럼 서버(텍스트)면 보통 SSH가 베스트예요.

지금 상황에 맞는 추천

지금 cp1의 Host-Only IP가 192.168.56.101이었죠.
바로 SSH로 붙어서 작업하면 복붙 스트레스가 확 줄어요.

원하면 제가:

cp1에서 SSH 설치

Windows에서 ssh 접속

netplan로 192.168.56.10 고정
까지 “복붙 가능한 방식”으로 한 번에 이어드릴게요.

지금 cp1 사용자명이 ubuntu 맞나요? (로그인 화면에 ubuntu로 보입니다)

Connection refused는 네트워크가 안 되는 게 아니라, cp1에서 SSH 서버(22번 포트)를 안 열어둔 상태일 때 99% 이렇게 뜹니다. (포트가 막힘이 아니라 “받아줄 프로세스가 없음”)

지금 cp1 콘솔에서 아래 순서대로 해주세요. 그대로 복붙하면 됩니다.

1) cp1에 SSH 서버 설치 + 실행

cp1(우분투)에서:

sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager -l


상태가 active (running)이면 OK.

2) 방화벽(UFW) 켜져 있으면 SSH 허용

대부분 기본은 꺼져있지만, 혹시 몰라서 확인:

sudo ufw status


Status: active면 아래 실행:

sudo ufw allow 22/tcp

3) cp1에서 22번 포트가 실제로 열렸는지 확인
sudo ss -lntp | grep ':22'


LISTEN이 보이면 SSH 준비 완료입니다.

4) Windows에서 다시 접속

PowerShell에서:

ssh ubuntu@192.168.56.101


처음이면 fingerprint 물어보는데 yes 입력.

5) (중요) VirtualBox 네트워크 구조상 주의

Windows(호스트) → cp1(Host-only) SSH는 Host-only IP(192.168.56.101) 로 접속하는 게 맞습니다. 지금 방향 OK.

만약 위를 했는데도 계속 refused면(바로 다음 체크)

cp1에서 이거 한 번만 보여주세요:

ip -br a
sudo systemctl status ssh --no-pager -l
sudo ss -lntp | grep ':22' || echo "22 not listening"


그러면 “ssh가 설치/실행이 안 된 건지” vs “다른 이유인지” 바로 갈라서 해결해드릴게요.

좋아요. 원인이 딱 나왔습니다.

ExecStartPre=/usr/sbin/sshd -t (status=1/FAILURE)
→ sshd 설정 검사(sshd -t) 단계에서 설정 오류가 있어서 SSH가 못 뜨는 상황이에요.
이럴 때는 보통 1) 호스트키 없음/권한 문제 2) sshd_config 문법 오류 3) 포트 충돌 중 하나입니다.

아래를 순서대로 그대로 실행해 주세요. (출력도 같이 보면 바로 확정됩니다)

1) 정확한 에러 메시지 확인
sudo sshd -t
echo $?


여기서 에러 문장이 반드시 뜹니다. (예: “Missing privilege separation directory…”, “Bad configuration option…”, “no hostkeys available…”)

그리고 로그도 같이:

sudo journalctl -xeu ssh --no-pager | tail -n 80

2) 가장 흔한 케이스: host key가 없어서 실패 (바로 복구)

아래 실행:

sudo ssh-keygen -A
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager -l

3) sshd 설정 파일 문법 체크 + 원복(필요시)

설정 파일을 건드린 적이 없다면 거의 없지만, 혹시를 위해:

sudo sshd -T | head


만약 sshd -t에서 “Bad configuration option” 같은 메시지가 나오면,
가장 안전한 원복:

sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%Y%m%d%H%M%S)
sudo dpkg-reconfigure openssh-server
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager -l

4) 포트 22가 다른 프로세스가 쓰는지 확인(드묾)
sudo ss -lntp | grep ':22' || echo "22 not listening"

5) SSH가 뜨면 Windows에서 다시 접속
ssh ubuntu@192.168.56.101

지금은 “2번(host key 생성)”이 가장 확률이 높습니다.

우선 이 3줄만 바로 실행해보세요:

sudo ssh-keygen -A
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager -l


그 결과(특히 status에 active (running) 뜨는지) 캡처/텍스트로 주시면, 바로 다음 단계(Windows에서 ssh 접속 성공 → netplan/hostname/클러스터 조인)로 이어갈게요.

원인 확정입니다. ✅
sshd: no hostkeys available -- exiting → SSH 호스트 키가 없어서 sshd가 시작을 못 하는 상태예요.

아래 3줄만 그대로 실행하면 해결됩니다.

sudo ssh-keygen -A
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager -l


정상이라면 status에 active (running)이 뜹니다.

그리고 Windows에서 다시 접속:

ssh ubuntu@192.168.56.101

(추가 확인이 필요하면)

cp1에서 22번 리슨 확인:

sudo ss -lntp | grep ':22'


먼저 위 3줄 실행 결과(특히 active (running) 여부)만 알려주세요.

좋아요. 지금 화면은 netplan 저장은 잘 됐고(“written”), sudo netplan apply도 실행된 상태입니다.

그리고 보이는 경고:

WARNING:root:Cannot call Open vSwitch: ovsdb-server.service is not running.

이건 대부분 무시해도 되는 경고예요. (Open vSwitch를 안 쓰는데도 netplan이 감지해서 말해주는 경우)
네트워크 적용 자체가 실패했다는 의미는 아닙니다.

1) 지금 제일 먼저 확인: IP가 바뀌었는지

아래를 실행해서 enp0s8이 원하는 IP로 바뀌었는지 확인하세요.

ip -br a


원하는 모습(예시):

enp0s8 UP 192.168.56.10/24

enp0s3 UP 10.0.2.15/24 (이건 그대로여도 OK)

만약 여전히 192.168.56.101이면, 설정이 덮어씌워지지 않았거나 YAML 들여쓰기/키가 다를 수 있어요.

2) netplan 설정 내용을 바로 보여주세요 (가장 확실)
sudo cat /etc/netplan/50-cloud-init.yaml


제가 그 내용을 보고 “정확히 어디가 잘못됐는지” 바로 잡아드릴게요.

3) 경고를 없애고 싶으면(선택)

Open vSwitch를 안 쓸 거면 그냥 무시가 정답이고,
굳이 없애려면 netplan 렌더러를 networkd로 명시하면 됩니다(파일에 따라 필요).

4) IP가 192.168.56.10으로 바뀌었다면 다음 단계
cp1 → Windows(Host-only) ping
ping -c 1 192.168.56.1

Windows → cp1 ping

PowerShell:

ping 192.168.56.10

지금은 “ip -br a” 출력이 제일 중요합니다

ip -br a 결과를 한 줄만 붙여주시면,

IP 변경 성공 여부

다음 단계(SSH 접속/클러스터 조인 준비)
바로 이어서 진행해드릴게요.