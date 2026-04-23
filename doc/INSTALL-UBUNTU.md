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

## AI 콜봇 (pyVoIP) 연동

### pjsip.conf — 콜봇 전용 엔드포인트

pyVoIP 기반 콜봇이 다른 서브넷 또는 VPN에 위치할 경우 아래 설정이 필수다.

```ini
[2001]
type=endpoint
transport=transport-udp
context=internal
disallow=all
allow=ulaw,alaw
auth=auth2001
aors=2001
; RTP를 Asterisk가 항상 중계 — 콜봇이 다른 서브넷/VPN에 있으므로 direct media 불가
direct_media=no
; SIP Contact 주소를 실제 수신 IP:Port로 보정 (NAT/VPN 환경)
force_rport=yes
rewrite_contact=yes
; rtp_symmetric=yes 금지: pyVoIP는 송수신 소켓을 분리(sin/sout)해서 사용함.
; rtp_symmetric=yes 설정 시 Asterisk가 sout의 임의 포트로 RTP를 보내 sin이 수신 못 함.

[auth2001]
type=auth
auth_type=userpass
username=2001
password=secret2001

[2001]
type=aor
max_contacts=1
remove_existing=yes
```

**핵심 설정 이유:**

| 설정 | 값 | 이유 |
|---|---|---|
| `direct_media` | `no` | 콜봇이 VPN/다른 서브넷 — direct RTP 불가, Asterisk가 중계해야 함 |
| `force_rport` | `yes` | NAT 뒤 콜봇의 실제 포트로 응답 전송 |
| `rewrite_contact` | `yes` | SIP Contact 헤더를 수신된 IP:Port로 덮어써 라우팅 오류 방지 |
| `rtp_symmetric` | (기본값 `no` 유지) | pyVoIP sin/sout 소켓 분리 구조 때문에 `yes`로 설정하면 RTP 수신 불가 |

### extensions.conf — 콜봇 내선 추가

```ini
[internal]
exten => 2001,1,Dial(PJSIP/2001,30)
 same => n,Hangup()
```

### 방화벽 — 콜봇 RTP 포트 허용

pyVoIP의 RTP 포트 범위를 좁게 설정해 방화벽 규칙을 명확히 관리한다.

```bash
# 콜봇 서버(172.31.90.1)의 RTP 포트 범위 허용
sudo ufw allow from 172.31.90.1 to any port 16000:16100 proto udp
```

콜봇 실행 시 환경변수로 포트 범위를 지정한다:
```bash
RTP_PORT_LOW=16000 RTP_PORT_HIGH=16100 python main.py
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

---

## 트러블슈팅: AI 콜봇 RTP 수신 불가 (STT 무음)

### 증상

TTS(콜봇 → 소프트폰) 는 정상이지만 STT(소프트폰 → 콜봇) 에 오디오가 수신되지 않는다.

```
[AudioSource] WARNING: No RTP audio received after 100 reads
[AudioSource] Done: 0/1248 reads had audio
```

### 원인: pyVoIP 소켓 비대칭 구조 + NAT 핀홀

pyVoIP는 RTP 송수신에 **별도 소켓**을 사용한다:

- `sout` (송신 전용): TTS 재생 시 Asterisk로 오디오 전송 → 패킷을 먼저 보내므로 NAT 핀홀이 자동 개통됨
- `sin` (수신 전용): STT를 위해 Asterisk로부터 오디오 수신 → **한 번도 패킷을 보내지 않으면 NAT 핀홀이 없어 수신 불가**

Stateful NAT/방화벽 환경(VPN 포함)에서 `sin` 방향 핀홀은 `sin`이 먼저 패킷을 보내야만 열린다.

### 해결: call.answer() 직후 sin 더미 패킷 전송

```python
call.answer()

# sin 소켓에서 Asterisk RTP 포트로 더미 패킷 전송 → 핀홀 개통
for rtp_client in call.RTPClients:
    try:
        rtp_client.sin.sendto(b'\x00', (rtp_client.outIP, rtp_client.outPort))
    except Exception as e:
        logger.warning(f"[RTP] Hole punch failed: {e}")
```

정상 동작 시 로그:
```
[RTP] Hole punch: sin(:16058) → 172.31.79.202:11520
[AudioSource] First audio received (160 bytes) after 1 reads
```

### 진단: RTP 패킷 도달 여부 확인

Asterisk 서버에서 tcpdump로 콜봇 방향 RTP 트래픽을 확인한다.

```bash
# Asterisk 서버에서 — 콜봇(172.31.90.1)으로 향하는 RTP 확인
sudo tcpdump -i any -n "host 172.31.90.1 and udp portrange 16000-16100"
```

패킷이 없으면 방화벽/라우팅 문제, 패킷이 있지만 콜봇에서 수신 못 하면 NAT 핀홀 문제다.

---

## 트러블슈팅: Google STT 인식 실패 (VAD 오동작)

### 증상 1 — transcript='timeout' (20초 대기 후 실패)

```
[STT] VAD: Speech BEGIN
[STT] VAD: Speech END / EOS
STT result: success=False, transcript='timeout'
```

**원인**: VAD END 이벤트 수신 시 `_stop = True`로 Google STT response 루프까지 즉시 종료했기 때문에 final transcript를 받지 못함.

**해결**: VAD END 시 audio 전송만 중단하고 response 루프는 계속 유지해 Google STT가 final transcript를 반환할 때까지 대기한다. 스트림이 종료됐는데도 결과가 없으면 즉시 `non_voice` 처리한다(20초 timeout 방지).

### 증상 2 — 실제 음성이 있는데 non_voice 처리

```
[AudioSource] Done: 401/549 reads had audio   ← 실제 음성 있음
[STT] Stream ended without final result → non_voice
```

**원인**: VAD END 직후 `_audio_source.stop()`을 즉시 호출해 Google STT가 buffered audio를 처리하기 전에 스트림이 닫힘.

**해결**: VAD END 후 즉시 스트림을 닫지 말고 ~500ms(4청크 × 125ms) 더 audio를 전송한 뒤 종료한다. Google STT가 남은 audio를 처리해 final transcript를 반환할 시간을 확보한다.

```
VAD END → 추가 4청크 전송(~500ms) → audio generator 종료
       → Google STT final transcript 반환 → 결과 수신
```

### 증상 3 — VAD BEGIN/END 반복 후 지연 인식

```
[STT] VAD: Speech BEGIN
[STT] VAD: Speech END / EOS
[STT] VAD: Speech BEGIN     ← 반복
[STT] VAD: Speech END / EOS
... (n회)
[STT] Final: '발화 내용'
```

**원인**: VAD END 이후 `_process_audio = False` 설정이 gRPC 스레드에 즉시 전파되지 않아 audio가 계속 전송됨.

**해결**: `_audio_generator`에서 `_process_audio` 플래그를 직접 체크해 EOS flush 이후 루프를 명시적으로 종료한다.
