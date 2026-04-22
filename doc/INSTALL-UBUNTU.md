# Asterisk 설치 가이드 (Ubuntu 18.04)

소프트폰 SIP REGISTER 및 내선 통화 테스트를 목적으로 한 설치 가이드입니다.

---

## 환경

- OS: Ubuntu 18.04.6 LTS
- 설치 방식: 소스 빌드 (Git 저장소 클론)
- SIP 스택: PJSIP (chan_pjsip)

---

## 1단계: 의존성 설치

```bash
sudo apt-get update && sudo apt-get upgrade -y

sudo apt-get install -y \
  build-essential git subversion \
  libssl-dev libncurses5-dev libnewt-dev \
  libxml2-dev libsqlite3-dev uuid-dev \
  libjansson-dev libedit-dev \
  pkg-config autoconf automake libtool
```

---

## 2단계: 소스 클론 및 빌드

```bash
cd ~
git clone https://github.com/<YOUR_FORK>/asterisk.git
cd asterisk

# 나머지 의존성 자동 설치
sudo contrib/scripts/install_prereq install

# 빌드 환경 구성 (jansson 내장 버전 사용 — 버전 충돌 방지)
./configure --with-jansson-bundled

# 모듈 선택 (chan_pjsip, res_pjsip* 활성화 확인)
make menuselect

# 병렬 빌드
make -j$(nproc)

# 시스템에 설치
sudo make install

# 샘플 설정 파일 설치 → /etc/asterisk/ 에 복사됨
sudo make samples

# systemd/init 서비스 등록
sudo make config

sudo ldconfig
```

> **경로 이해**
> - `~/asterisk/` — 소스코드 디렉토리 (빌드 전용)
> - `/etc/asterisk/` — 런타임 설정 디렉토리 (Asterisk가 실행 시 읽는 곳)
> - `/usr/sbin/asterisk` — 설치된 실행 바이너리

---

## 3단계: SIP 설정

### `/etc/asterisk/pjsip.conf`

```bash
sudo tee /etc/asterisk/pjsip.conf << 'EOF'
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0:5060

; ── 내선 1001 ──────────────────────────────────────
[1001]
type=endpoint
transport=transport-udp
context=internal
disallow=all
allow=ulaw,alaw,g722
auth=auth1001
aors=1001

[auth1001]
type=auth
auth_type=userpass
username=1001
password=secret1001

[1001]
type=aor
max_contacts=1
remove_existing=yes

; ── 내선 1002 ──────────────────────────────────────
[1002]
type=endpoint
transport=transport-udp
context=internal
disallow=all
allow=ulaw,alaw,g722
auth=auth1002
aors=1002

[auth1002]
type=auth
auth_type=userpass
username=1002
password=secret1002

[1002]
type=aor
max_contacts=1
remove_existing=yes
EOF
```

### `/etc/asterisk/extensions.conf`

```bash
sudo tee /etc/asterisk/extensions.conf << 'EOF'
[internal]
exten => 1001,1,Dial(PJSIP/1001,30)
 same => n,Hangup()

exten => 1002,1,Dial(PJSIP/1002,30)
 same => n,Hangup()

; 에코 테스트 (9999로 전화하면 자기 목소리가 들림)
exten => 9999,1,Answer()
 same => n,Echo()
 same => n,Hangup()
EOF
```

---

## 4단계: 로그 설정

### `/etc/asterisk/logger.conf`

```bash
sudo tee /etc/asterisk/logger.conf << 'EOF'
[general]

[logfiles]
console => notice,warning,error,verbose
full => notice,warning,error,debug,verbose,dtmf
EOF
```

---

## 5단계: Asterisk 시작

```bash
# 서비스 시작 및 부팅 시 자동 시작 등록
sudo systemctl start asterisk
sudo systemctl enable asterisk

# CLI 접속
sudo asterisk -r
```

---

## 6단계: 동작 확인 (CLI)

```
# 엔드포인트 확인 (소프트폰 REGISTER 전: Unavailable)
tts-dev-001*CLI> pjsip show endpoints

# 트랜스포트 확인
tts-dev-001*CLI> pjsip show transports

# 활성 채널 확인
tts-dev-001*CLI> core show channels

# SIP 패킷 실시간 출력 (디버깅용)
tts-dev-001*CLI> pjsip set logger on

# 로거 재시작 (logger.conf 변경 후)
tts-dev-001*CLI> logger reload
```

정상 상태 출력 예시:

```
Endpoint:  1001    Unavailable   0 of inf
Endpoint:  1002    Unavailable   0 of inf
Transport: transport-udp   udp   0.0.0.0:5060
```

---

## 7단계: 로그 확인

```bash
# 실시간 전체 로그
sudo tail -f /var/log/asterisk/full

# 에러/경고만 필터
sudo tail -f /var/log/asterisk/full | grep -E "ERROR|WARNING"
```

---

## 8단계: 소프트폰 연결

Zoiper, Linphone, MicroSIP 등에서 아래 정보로 SIP 계정 추가:

| 항목 | 1001 계정 | 1002 계정 |
|------|-----------|-----------|
| SIP Server | `172.31.79.202` | `172.31.79.202` |
| Username | `1001` | `1002` |
| Password | `secret1001` | `secret1002` |
| Port | `5060` | `5060` |
| Transport | UDP | UDP |

REGISTER 성공 시 CLI에서 상태 변화 확인:

```
Endpoint:  1001    Available   0 of inf   ← Unavailable → Available
```

**통화 테스트:**
- 1001 → 1002 전화 걸기
- 에코 테스트: `9999` 로 전화

---

## 방화벽 설정

```bash
sudo ufw allow 5060/udp         # SIP 시그널링
sudo ufw allow 10000:20000/udp  # RTP 미디어

sudo ufw status
```

---

## 트러블슈팅: 소프트폰 REGISTER 실패

### 1. 패킷이 서버에 도달하는지 확인

서버에서 tcpdump로 모니터링하면서 클라이언트에서 UDP 패킷을 전송해 도달 여부를 확인한다.

**서버:**
```bash
sudo tcpdump -i any -n port 5060 -v
```

**클라이언트 (Mac):**
```bash
# 실행 후 텍스트 입력 → 서버 tcpdump에 패킷이 찍히면 UDP 통신 가능
nc -u <서버_IP> 5060
```

패킷이 서버에 도달하지 않으면 VPN/라우팅 문제이며, 도달하면 Asterisk 설정 또는 소프트폰 설정 문제다.

### 2. Asterisk SIP 로거 활성화

소프트폰 REGISTER 시도 시 SIP 패킷 내용을 실시간으로 확인한다.

```
tts-dev-001*CLI> pjsip set logger on
```

### 3. 소프트폰 설정에서 접속 IP 확인

에러 메시지나 소프트폰 계정 정보에서 실제로 어떤 IP로 접속 시도 중인지 확인한다.

```
# Zoiper 에러 예시
Account name: 1001@172.31.80.19   ← 이 IP가 올바른지 확인
```

잘못된 IP가 설정되어 있으면 Asterisk 서버 IP(`172.31.79.202`)로 수정 후 재등록한다.

### 4. 정상 REGISTER SIP 흐름

아래 흐름이 tcpdump에 찍히면 REGISTER 성공이다.

```
클라이언트 →  REGISTER          →  Asterisk
클라이언트 ←  401 Unauthorized  ←  Asterisk  (인증 챌린지, 정상)
클라이언트 →  REGISTER + 인증   →  Asterisk
클라이언트 ←  200 OK            ←  Asterisk  ✓ 등록 완료
```

CLI에서 상태 변화 확인:
```
tts-dev-001*CLI> pjsip show endpoints
Endpoint:  1001    Available   0 of inf   ← Unavailable → Available
```
