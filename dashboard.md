좋아요. 그럼 지금 만든 K3s 멀티노드(ubuntu 22.04, VirtualBox Host-Only 192.168.56.x) 기준으로, 바로 이어서 **Ingress(웹 라우팅) + Metrics Server(모니터링) + Kubernetes Dashboard(웹 UI)**까지 “학습용 세트”로 붙여드릴게요.
(K3s는 기본 패키지 컴포넌트로 Traefik(ingress) + metrics-server를 함께 배포하는 구성이 일반적입니다. 
K3s
+1
)

1) 현재 상태 확인 (cp1에서)
kubectl get nodes -o wide
kubectl -n kube-system get deploy,svc | egrep "traefik|metrics"


K3s의 Traefik은 보통 80/443 포트를 쓰는 LoadBalancer 서비스로 올라옵니다. (ServiceLB가 노드들에 노출) 
SUSE Documentation
+1

Traefik 서비스가 어디로 노출되는지 확인:

kubectl -n kube-system get svc traefik -o wide

2) Ingress 실습 (기본 Traefik로 진행)
2-1) 테스트 앱(whoami) 배포
kubectl create deploy whoami --image=traefik/whoami
kubectl expose deploy whoami --port 80
kubectl get pod -o wide

2-2) Ingress 생성 (도메인 기반 라우팅)

아래를 ing-whoami.yaml로 저장 후 적용:

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-ing
spec:
  rules:
  - host: whoami.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: whoami
            port:
              number: 80


적용:

kubectl apply -f ing-whoami.yaml
kubectl get ingress

2-3) Windows hosts 파일에 도메인 매핑

Windows(관리자 권한)에서 아래 파일 열기:

C:\Windows\System32\drivers\etc\hosts

맨 아래에 추가:

192.168.56.10  whoami.local


이후 브라우저에서:

http://whoami.local/

만약 안 열리면, 우선 IP로도 테스트해보세요: http://192.168.56.10/
(Traefik이 80 포트로 노출되는 구조라 포트 충돌이 있으면 다른 서비스가 80/443를 못 쓸 수 있습니다. 
SUSE Documentation
)

3) Metrics Server(모니터링) 확인 & 사용
3-1) “top” 명령으로 확인
kubectl top nodes
kubectl top pods -A | head


K3s는 패키지 컴포넌트로 metrics-server를 포함할 수 있고(기본 애드온), 이게 떠 있어야 kubectl top이 동작합니다. 
K3s

3-2) 만약 metrics not available가 나오면 (대응)

metrics-server가 떠 있는지:

kubectl -n kube-system get deploy metrics-server
kubectl -n kube-system logs deploy/metrics-server --tail=50


그래도 해결이 안 되면(고급/최후수단):
K3s 패키지 매니페스트는 재기동 시 다시 써져서 직접 수정하면 깨질 수 있어, 패키지 metrics-server를 disable하고 upstream metrics-server를 별도 설치하는 방식이 권장됩니다. 
GitHub
+1

Upstream 설치는 metrics-server 공식이 다음 한 줄입니다. 
GitHub
+1

kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml


(필요하면 제가 “K3s에서 metrics-server disable → upstream로 교체”까지 정확히 시스템 서비스/설정파일 기준으로 이어서 잡아드릴게요.)

4) Kubernetes Dashboard(웹 UI) 설치 + 접속

Kubernetes 공식 문서 기준으로 Dashboard는 Helm 설치가 표준입니다. 
Kubernetes
+1

4-1) Helm 설치 (cp1에서)
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

4-2) Dashboard 설치
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace --namespace kubernetes-dashboard


Kubernetes

4-3) Dashboard 접속 (포트포워딩)

cp1에서 실행:

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443


Kubernetes

cp1에서 브라우저면: https://localhost:8443

Windows에서 보고 싶으면(권장X, 학습용만): 아래처럼 바인딩

kubectl -n kubernetes-dashboard port-forward --address 0.0.0.0 svc/kubernetes-dashboard-kong-proxy 8443:443


그 뒤 Windows 브라우저에서: https://192.168.56.10:8443

4-4) 로그인 토큰 만들기 (학습용 “관리자”)

공식 문서도 “샘플 유저는 관리자 권한이라 교육용”이라고 경고합니다. 
Kubernetes

학습 편의를 위해 아래로 만드세요.

dash-admin.yaml:

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard


적용 + 토큰 출력:

kubectl apply -f dash-admin.yaml
kubectl -n kubernetes-dashboard create token admin-user


나오는 토큰을 Dashboard 로그인 화면에 붙여넣으면 됩니다.

다음 “학습 미션” (진짜 실력 느는 순서)

Ingress로 whoami → nginx 두 서비스 분기 라우팅

kubectl top으로 리소스 보고, requests/limits 걸어서 스케줄링 변화를 관찰

Dashboard에서 Deployment scale/rollout/로그 확인까지 한 바퀴

원하시면, 지금 상황에 맞게 제가 **“Windows에서 whoami.local이 안 열릴 때 점검 순서(방화벽/포트/Traefik 서비스 타입/주소 확인)”**를 체크리스트로 바로 드릴게요. 지금 whoami까지 진행하면 **어느 단계에서 막혔는지(응답 없음 / 404 / 인증서 경고 / 다른 페이지 뜸)**만 짧게 알려주세요.


에러 메시지 핵심은 이거예요:

Get "http://localhost:8080/version": connect: connection refused
→ Helm이 Kubernetes API를 어디로 접속해야 하는지(kubeconfig)를 못 찾아서, 기본값인 localhost:8080으로 가다가 실패한 겁니다.
(= 클러스터가 없다는 뜻이 아니라, Helm이 클러스터 설정을 모른다는 뜻)

아래 순서대로 하면 바로 해결됩니다.

1) 먼저 k3s가 정상인지 확인 (cp1에서)
sudo systemctl status k3s --no-pager -l
sudo kubectl get nodes -o wide


Ready가 보이면 클러스터는 살아있는 겁니다.

2) Helm이 k3s kubeconfig를 보도록 설정 (가장 간단)

cp1에서:

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
helm ls -A


이제 Helm이 클러스터를 찾을 수 있어야 합니다.

3) 영구적으로 설정(추천)
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
source ~/.bashrc

4) 권한 문제 나면(읽기 권한 없을 때)

k3s 설치 때 K3S_KUBECONFIG_MODE=644 안 줬으면 일반 유저가 파일을 못 읽을 수 있어요. 그럴 땐 내 홈으로 복사해서 쓰면 됩니다.

mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
unset KUBECONFIG
kubectl get nodes

5) 다시 Dashboard 설치

(위 설정이 된 상태에서)

helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace --namespace kubernetes-dashboard

빠른 원인 요약

Helm은 kubeconfig가 없으면 기본값 localhost:8080 로 가요.

k3s의 kubeconfig는 보통 /etc/rancher/k3s/k3s.yaml 에 있습니다.

원하면, sudo kubectl get nodes -o wide 출력만 붙여주세요.
그걸 보고 “클러스터 상태 + 다음으로 대시보드 접속(포트포워딩/Ingress/Traefik)”까지 바로 이어서 안내해드릴게요.

ubuntu@cp1:~$ echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
ubuntu@cp1:~$ source ~/.bashrc
ubuntu@cp1:~$ helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace --namespace kubernetes-dashboard
Release "kubernetes-dashboard" does not exist. Installing it now.
NAME: kubernetes-dashboard
LAST DEPLOYED: Sun Jan  4 02:59:20 2026
NAMESPACE: kubernetes-dashboard
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*************************************************************************************************
*** PLEASE BE PATIENT: Kubernetes Dashboard may need a few minutes to get up and become ready ***
*************************************************************************************************

Congratulations! You have just installed Kubernetes Dashboard in your cluster.

To access Dashboard run:
  kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

NOTE: In case port-forward command does not work, make sure that kong service name is correct.
      Check the services in Kubernetes Dashboard namespace using:
        kubectl -n kubernetes-dashboard get svc

Dashboard will be available at:
  https://localhost:8443
ubuntu@cp1:~$

좋습니다. ✅ 대시보드가 정상 설치(deployed) 됐어요. 이제 “접속”만 하면 됩니다. 보통 2가지 루트가 있어요: (A) cp1에서 직접 포트포워딩 / (B) Windows(호스트)에서 포트포워딩.

A) cp1에서 바로 접속(가장 간단)

cp1 터미널에서:

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443 --address 0.0.0.0


그 다음 Windows 브라우저에서:

https://192.168.56.10:8443
(※ cp1의 Host-only IP가 192.168.56.10이 아니라면, 현재 cp1의 IP로 바꿔서 접속)

브라우저에서 “안전하지 않음” 경고가 뜨면 고급 → 계속으로 들어가면 됩니다(자체 인증서).

B) Windows에서 포트포워딩(SSH로 터널링, 더 깔끔)

Windows PowerShell에서:

ssh -L 8443:localhost:8443 ubuntu@192.168.56.10


그리고 cp1(SSH 세션) 안에서 포트포워딩 실행:

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443


이제 Windows 브라우저에서:

https://localhost:8443

로그인 토큰 만들기 (필수)

Dashboard는 기본적으로 토큰이 있어야 로그인됩니다. 아래를 cp1에서 실행하세요.

1) 관리자용 ServiceAccount 생성
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

2) 토큰 출력

(K8s 버전에 따라 이게 제일 안정적)

kubectl -n kubernetes-dashboard create token admin-user


나오는 토큰을 **Dashboard 로그인 화면(Token)**에 붙여넣으면 됩니다.

상태 확인(대시보드 Pod이 준비됐는지)
kubectl -n kubernetes-dashboard get pods,svc


원하시는 접속 방식이 뭐예요?

Windows에서 https://localhost:8443 로 보고 싶다 → (B) 추천

바로 https://192.168.56.10:8443 로 보고 싶다 → (A)로 가면 됩니다

지금 cp1의 Host-only IP가 확정됐나요(192.168.56.10으로 바꾸셨나요)? 아니라면 ip -br a 한 줄만 보내주시면 접속 URL을 정확히 찍어드릴게요.

ubuntu@cp1:~$ export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
ubuntu@cp1:~$ helm ls -A
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                           APP VERSION
traefik         kube-system     1               2026-01-03 02:09:34.162445981 +0000 UTC deployed        traefik-37.1.1+up37.1.0         v3.5.1
traefik-crd     kube-system     1               2026-01-03 02:09:21.734531284 +0000 UTC deployed        traefik-crd-37.1.1+up37.1.0     v3.5.1
ubuntu@cp1:~$ echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
ubuntu@cp1:~$ source ~/.bashrc
ubuntu@cp1:~$ helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  --create-namespace --namespace kubernetes-dashboard
Release "kubernetes-dashboard" does not exist. Installing it now.
NAME: kubernetes-dashboard
LAST DEPLOYED: Sun Jan  4 02:59:20 2026
NAMESPACE: kubernetes-dashboard
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
*************************************************************************************************
*** PLEASE BE PATIENT: Kubernetes Dashboard may need a few minutes to get up and become ready ***
*************************************************************************************************

Congratulations! You have just installed Kubernetes Dashboard in your cluster.

To access Dashboard run:
  kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

NOTE: In case port-forward command does not work, make sure that kong service name is correct.
      Check the services in Kubernetes Dashboard namespace using:
        kubectl -n kubernetes-dashboard get svc

Dashboard will be available at:
  https://localhost:8443
ubuntu@cp1:~$ kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443 --address 0.0.0.0
Forwarding from 0.0.0.0:8443 -> 8443

Handling connection for 8443
Handling connection for 8443
Handling connection for 8443
Handling connection for 8443
Handling connection for 8443
^Cubuntu@cp1:~ssh -L 8443:localhost:8443 ubuntu@192.168.56.1010
The authenticity of host '192.168.56.10 (192.168.56.10)' can't be established.
ED25519 key fingerprint is SHA256:zRAMiX2cfcZyQBYiCwXdqpIMm2jbG9UJrFUs3qFc4hY.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.56.10' (ED25519) to the list of known hosts.
ubuntu@192.168.56.10's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-164-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
New release '24.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Sun Jan  4 02:51:55 2026 from 192.168.56.1
ubuntu@cp1:~$ kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
Unable to listen on port 8443: Listeners failed to create with the following errors: [unable to create listener: Error listen tcp4 127.0.0.1:8443: bind: address already in use unable to create listener: Error listen tcp6 [::1]:8443: bind: address already in use]
error: unable to listen on any of the requested ports: [{8443 8443}]
ubuntu@cp1:~$ kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
Unable to listen on port 8443: Listeners failed to create with the following errors: [unable to create listener: Error listen tcp4 127.0.0.1:8443: bind: address already in use unable to create listener: Error listen tcp6 [::1]:8443: bind: address already in use]
error: unable to listen on any of the requested ports: [{8443 8443}]
ubuntu@cp1:~$

좋아요. 지금 상황은 단순합니다.

Windows에서 ssh -L 8443:localhost:8443 ... 로 로컬 8443 포트를 이미 점유했거나,

cp1 안에서 이전 port-forward가 남아 있어서 8443이 “이미 사용 중(address already in use)” 입니다.

그래서 kubectl port-forward ... 8443:443가 다시 뜨려다가 실패한 거예요.

아래 중 하나만 하면 바로 해결됩니다. (추천은 1번)

1) 포트 번호 바꿔서 바로 실행 (가장 빠름)

cp1(SSH 접속된 세션)에서 다른 포트로 포워딩하세요. 예: 9443

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 9443:443


그 다음 브라우저:

(SSH 터널 없이면) https://192.168.56.10:9443

(SSH 터널을 쓰려면) Windows에서 -L 9443:localhost:9443로 바꿔야 함

2) “누가 8443 쓰는지” 확인하고 죽이기 (정석)

cp1에서:

sudo ss -lntp | grep ':8443' || echo "no listener on 8443"
sudo lsof -iTCP:8443 -sTCP:LISTEN -n -P


나오는 PID를 종료:

sudo kill <PID>
# 안 죽으면
sudo kill -9 <PID>


그 후 다시:

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443

3) 아까 --address 0.0.0.0로 띄웠던 세션이 아직 살아있을 수 있음

그 세션을 Ctrl+C로 끊긴 것 같긴 한데(로그에 ^C가 보임), 혹시 백그라운드/다른 터미널에 남아있으면 2번 방식으로 정리하면 됩니다.

그리고 SSH 터널 명령이 “실행 위치”가 조금 꼬였어요

ssh -L ... 는 Windows PowerShell에서 실행해야 하고,
방금 로그를 보면 cp1 프롬프트에서 실행한 흔적이 있어요(ubuntu@cp1:~ssh -L ... 처럼).

원하는 패턴이 이거라면:

✅ 권장 패턴(깔끔)

Windows PowerShell:

ssh -L 9443:localhost:9443 ubuntu@192.168.56.10


SSH로 들어간 cp1 터미널:

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 9443:443


Windows 브라우저:

https://localhost:9443

지금 바로 해보실 “최단 코스”

cp1에서 이 한 줄만:

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 9443:443


그리고 Windows에서:

https://192.168.56.10:9443 로 접속해보세요.

접속되면 다음은 토큰 생성입니다:

kubectl -n kubernetes-dashboard create token admin-user


원하면, 지금 cp1에서 sudo ss -lntp | grep 8443 결과를 붙여주시면 “누가 점유 중인지”도 바로 찍어드릴게요.

ubuntu@cp1:~$ sudo ss -lntp | grep ':8443' || echo "no listener on 8443"
[sudo] password for ubuntu:
LISTEN 0      128        127.0.0.1:8443       0.0.0.0:*    users:(("ssh",pid=7016,fd=5))
LISTEN 0      128            [::1]:8443          [::]:*    users:(("ssh",pid=7016,fd=4))
ubuntu@cp1:~$ sudo lsof -iTCP:8443 -sTCP:LISTEN -n -P
sudo: lsof: command not found

딱 원인 나왔어요 ✅

127.0.0.1:8443 를 ssh(pid=7016) 가 잡고 있습니다.
→ 즉, SSH 로컬 포워딩(-L) 이 이미 8443을 점유 중이라서 kubectl port-forward가 8443에 못 뜨는 겁니다.

그리고 lsof는 아직 설치 안 돼서 command not found가 뜬 거고요.

해결 방법 2가지 (둘 중 하나만)
방법 A (추천): 포트 번호를 바꿔서 충돌 회피

SSH 터널도, kubectl 포워딩도 9443 같은 다른 포트로 바꾸면 깔끔합니다.

지금 ssh(pid=7016) 터널 세션을 끊고(Windows에서 실행한 그 ssh 창에서 exit 또는 창 닫기)

Windows PowerShell에서:

ssh -L 9443:localhost:9443 ubuntu@192.168.56.10


cp1(SSH 접속 안)에서:

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 9443:443


Windows 브라우저:

https://localhost:9443

방법 B: 지금 점유 중인 8443 SSH 포워딩을 종료

cp1에서 보이는 ssh pid=7016을 종료하면 8443이 비어요.

sudo kill 7016


그 다음 다시:

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443


단, 이러면 SSH 터널이 끊길 수 있으니(세션/포워딩 구조에 따라) 보통은 A처럼 포트를 바꾸는 게 편합니다.

lsof 설치(선택)

다음부터 포트 점유 확인을 쉽게 하려면:

sudo apt update
sudo apt install -y lsof

추가 팁: SSH 터널 없이도 접속 가능

아까처럼 cp1에서 아래로 띄우면(바인드 0.0.0.0):

kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443 --address 0.0.0.0


Windows 브라우저에서:

https://192.168.56.10:8443

이 방식이면 SSH 터널(-L) 자체가 필요 없습니다.

원하는 방식이 어느 쪽이에요?

(1) SSH 터널 없이 https://192.168.56.10:8443 로 접속

(2) SSH 터널로 https://localhost:9443 로 접속 (추천: PC에서 작업 편함)

원하시는 쪽에 맞춰 “토큰 생성 + 로그인”까지 한 번에 이어드릴게요.

![alt text](image-3.png)