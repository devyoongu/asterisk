# 원격 Asterisk 로그 보기

배포된 Asterisk PBX 서버(`172.31.79.202`, Ubuntu)에 원격으로 붙어 로그를
확인하는 방법 모음.

> 서버 IP / 사용자 / SSH 키는 환경마다 다르므로 아래 예시의 `<USER>`, `<HOST>`
> 자리는 본인 환경에 맞게 치환하세요. 이 저장소의 기본 배포 환경은
> `<HOST>=172.31.79.202` 입니다.

---

## 1. SSH 접속

```bash
ssh <USER>@<HOST>
# 예: ssh ubuntu@172.31.79.202
```

이후 명령은 모두 SSH 세션 안에서 실행한다고 가정합니다. 한 번에 끝낼 수 있는
명령은 `ssh <USER>@<HOST> "<command>"` 형태로 로컬에서 바로 실행해도 됩니다.

---

## 2. 로그 파일 직접 보기 (`/var/log/asterisk/full`)

기본 로그 경로와 채널은 `/etc/asterisk/logger.conf` 에서 정의됨
(`docs/INSTALL-UBUNTU.md` 4단계 참고):

```ini
[logfiles]
console => notice,warning,error,verbose
full    => notice,warning,error,debug,verbose,dtmf
```

### 실시간 tail

```bash
# 전체 실시간
sudo tail -f /var/log/asterisk/full

# 에러/경고만
sudo tail -f /var/log/asterisk/full | grep -E "ERROR|WARNING"

# 특정 익스텐션(예: 콜봇 2001)만
sudo tail -f /var/log/asterisk/full | grep -E "2001|PJSIP/2001"

# 특정 콜 ID로 추적 (CLI에서 콜 ID 확인 후)
sudo tail -f /var/log/asterisk/full | grep -i "<call-id>"
```

### 과거 로그 검색

```bash
# 마지막 N줄
sudo tail -n 500 /var/log/asterisk/full

# 시각 범위로 보기 (less + 검색)
sudo less /var/log/asterisk/full
# less 안에서 / 로 검색, F 로 tail -f 모드

# 회전된 로그까지 합쳐서 검색
sudo zgrep -h "ERROR" /var/log/asterisk/full*
```

### 로컬로 받아서 분석

```bash
# 로컬에서 실행 (콜 한 건 분량을 받아 IDE/그렙으로 분석할 때)
scp <USER>@<HOST>:/var/log/asterisk/full ./asterisk-full.log
```

---

## 3. 원격 CLI 접속 (`asterisk -r`)

대화형 CLI에 붙어 실시간 verbose 출력을 보고 명령도 실행할 수 있습니다.

```bash
# 서버에서
sudo asterisk -rvvv      # v 개수 = verbose 레벨
# 종료: exit  또는  Ctrl-D
```

CLI에서 자주 쓰는 진단 명령:

```
core show channels         ; 활성 콜 채널
pjsip show endpoints       ; 엔드포인트 등록 상태 (1001/1002/1003/2001)
pjsip show contacts        ; 실제 등록된 컨택트 (IP/포트 포함)
pjsip show registrations   ; 외부로 register 중인 항목
core show uptime           ; 가동 시간
logger show channels       ; 현재 활성 로그 채널/레벨
```

원격에서 한 줄만 실행하고 싶을 때:

```bash
ssh <USER>@<HOST> "sudo asterisk -rx 'pjsip show endpoints'"
ssh <USER>@<HOST> "sudo asterisk -rx 'core show channels'"
```

---

## 4. SIP / PJSIP 패킷 트레이스

콜 시그널링 문제(REGISTER 실패, INVITE 라우팅 등)를 볼 때 가장 유용합니다.

```
*CLI> pjsip set logger on            ; SIP 패킷 로깅 켜기
*CLI> pjsip set logger host 1.2.3.4  ; 특정 IP 와의 패킷만
*CLI> pjsip set logger off
```

켜져 있는 동안 모든 SIP 메시지가 console / `full` 로그에 찍힙니다. 잡음이
많으니 디버깅이 끝나면 반드시 `off`.

RTP 쪽까지 보고 싶으면:

```
*CLI> rtp set debug on
*CLI> rtp set debug off
```

---

## 5. 모듈별 디버그 레벨 일시 상승

재시작 없이 verbose / debug 를 올렸다가 내릴 수 있습니다.

```
*CLI> core set verbose 5
*CLI> core set debug 5
*CLI> core set debug 5 res_pjsip      ; 특정 모듈만
*CLI> core set verbose 0              ; 원복
*CLI> core set debug 0
```

`logger.conf` 를 영구 변경했다면:

```
*CLI> logger reload
```

---

## 6. systemd 서비스 로그 (`journalctl`)

Asterisk 데몬 자체의 시작/실패/시그널은 systemd 저널에 남습니다.
(애플리케이션 로그는 위 `full` 쪽이 더 상세함 — 이쪽은 "왜 안 떴지?" 때 봄)

```bash
# 최근 부트 이후
sudo journalctl -u asterisk -b

# 실시간
sudo journalctl -u asterisk -f

# 시간 범위
sudo journalctl -u asterisk --since "1 hour ago"
sudo journalctl -u asterisk --since "2026-05-08 09:00" --until "2026-05-08 10:00"
```

서비스 상태 한 번에 확인:

```bash
sudo systemctl status asterisk
```

---

## 7. 자주 쓰는 일회성 진단 (로컬에서 바로)

```bash
# 등록 상태 한 번 확인
ssh <USER>@<HOST> "sudo asterisk -rx 'pjsip show endpoints'"

# 콜봇(2001) 컨택트 IP/포트 확인 — VPN/NAT 환경 디버깅용
ssh <USER>@<HOST> "sudo asterisk -rx 'pjsip show contacts' | grep 2001"

# 최근 5분 로그 중 콜봇 관련만
ssh <USER>@<HOST> "sudo grep 'PJSIP/2001' /var/log/asterisk/full | tail -n 200"
```

---

## 참고

- 로그 설정 / 경로 정의: `docs/INSTALL-UBUNTU.md` 4단계, 7단계
- 등록 성공 로그 샘플: `callbot/docs/troubleshooting.md`
- 배포 중인 PJSIP / 다이얼플랜 설정: `asterisk/conf/pjsip.conf`,
  `asterisk/conf/extensions.conf`
