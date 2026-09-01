# 🐈 Tailcat 완전 정복 가이드 (한국어)

> Tailcat 저장소를 분석하고, 직접 빌드·실행해보면서 정리한 문서입니다.
> 설치법부터 사용법, PHP 연동 가능성, 수익화 아이디어까지 담았습니다.

## 📌 저장소 주소

| 구분 | 주소 |
| --- | --- |
| 내 저장소 (fork/upload) | https://github.com/bmshin94/tailcat |
| 원본 저장소 (Tailscale 공식) | https://github.com/tailscale/tailcat |
| 릴리즈 (완제품 다운로드) | https://github.com/tailscale/tailcat/releases |
| 컨테이너 이미지 | https://github.com/tailscale/tailcat/pkgs/container/tailcat |
| 브라우저 웹 데모 | https://tailscale.github.io/tailcat/ |
| Go 패키지 문서 | https://pkg.go.dev/github.com/tailscale/tailcat |
| 기본 DERP 릴레이 목록 | https://tailcat.dev/derpmap.json |
| DERP 서버 직접 운영하기 | https://github.com/tailscale/tailscale/tree/main/cmd/derper#derp |

---

## 목차

1. [Tailcat이 뭔가요?](#1-tailcat이-뭔가요)
2. [폴더 구조 분석](#2-폴더-구조-분석)
3. [설치하기](#3-설치하기)
4. [사용법](#4-사용법)
5. [열쇠(Key) 관리](#5-열쇠key-관리)
6. [동작 원리](#6-동작-원리)
7. [PHP로 만들 수 있나요?](#7-php로-만들-수-있나요)
8. [수익화 아이디어](#8-수익화-아이디어)
9. [주의사항](#9-주의사항)
10. [검증 기록](#10-검증-기록)

---

## 1. Tailcat이 뭔가요?

한 줄 요약: **계정도, 설치도, 관리자 권한도 없이 두 컴퓨터를 암호화된 직통 터널로 연결해주는 도구.**

만든 곳은 VPN으로 유명한 **Tailscale**. README에 있는 슬로건이 성격을 잘 설명합니다.

> *"Tailscale without Tailscale, by Tailscale"*

원래 Tailscale은 계정 로그인 → 중앙 관리 서버(control plane)를 거쳐야 합니다.
Tailcat은 그 **관리 서버 부분만 걷어낸** 버전이고, 대신 연결 정보를
**토큰 문자열 하나**로 직접 주고받습니다.

```
🐈 Server listening with new address: tcomFwWCCcjS5nKNqAod034nWoJZW0LZ...
```

이 `tc...` 로 시작하는 문자열이 **주소이자 비밀번호**입니다.
상대방에게 전달하면 연결 끝.

### 기존 방식과 비교

```
😩 기존:     내 PC  →  카톡/구글드라이브(남의 창고)  →  상대 PC
✨ Tailcat:  내 PC  ⟷⟷⟷ (직통 암호화 터널) ⟷⟷⟷  상대 PC
```

### 핵심 특징

| 항목 | 내용 |
| --- | --- |
| 💰 비용 | 무료 (오픈소스) |
| 👤 계정 | 필요 없음 |
| 🔒 보안 | WireGuard 종단간 암호화 |
| 🎫 주소 | 기본이 일회용 (프로그램 종료 시 소멸) |
| 🪶 설치 | 실행파일 하나, 루트 권한 불필요 |
| 🌐 설정 | 라우팅/DNS 등 시스템 설정 안 건드림 |
| 📜 라이선스 | BSD 3-Clause |

---

## 2. 폴더 구조 분석

Go 언어 프로젝트이며, 테스트 코드가 촘촘히 붙어 있습니다.

| 위치 | 역할 |
| --- | --- |
| `tailcat.go` (1,954줄) | **핵심 엔진.** 터널 생성, 피어 탐색, 연결 관리 |
| `cmd/tailcat/` (1,675줄) | **CLI 도구.** `serve`, `cp`, `ls`, `ssh`, `ping`, `socks` 등 |
| `disco.go` | **NAT 홀펀칭.** 공유기 뒤에 있어도 서로 찾아내는 부분 |
| `tailcat_sftp.go` | 파일 전송(SFTP) 서비스 |
| `tailcat_ssh*.go` | SSH 서버 (OS별 구현 분리: unix / windows / stub) |
| `wire.go` | 연결 토큰(ConnBlob) 인코딩/디코딩 |
| `pickregion.go` | 가장 가까운 DERP 릴레이 지역 선택 |
| `web/`, `webdemo/` | WebAssembly로 컴파일한 브라우저 데모 |
| `internal/` | 빌드 태그 관리 등 내부 유틸 |
| `tool/updateflakes/` | Nix flake 해시 자동 갱신 도구 |
| `.github/workflows/` | CI 자동화 (테스트, 릴리즈, GitHub Pages 배포) |
| `*_test.go` | 단위 / 통합 / E2E 테스트 |
| `go.mod` | Go 1.27, `tailscale.com` 의존 |
| `build-tags.txt` | 공식 빌드용 태그 목록 (바이너리 약 16% 축소) |
| `LICENSE` | BSD 3-Clause |

> 🐱 **귀여운 포인트:** 두 컴퓨터가 처음 인사하는 메시지 이름이 실제로
> **`Meow`(야옹)** 과 **`Meowed`(야옹했음)** 입니다.

### 역사

- 2023년 9월, 장거리 비행 중 "derpcat"이라는 이름으로 시작
- 처음엔 tailscale 저장소의 fork 안에 있었고 여러 번 방치됨
- 이후 독립된 Go 모듈로 리팩터링
- **2026년 8월 TailscaleUp 컨퍼런스에서 오픈소스로 공개**

---

## 3. 설치하기

### 방법 1: macOS — Homebrew (가장 쉬움)

```bash
brew install tailcat
```

### 방법 2: 완제품 다운로드

https://github.com/tailscale/tailcat/releases

| OS | 파일 |
| --- | --- |
| Linux | `.tar.gz` (정적 바이너리) |
| Ubuntu/Debian | `.deb` → `sudo dpkg -i tailcat*.deb` |
| RedHat/Fedora | `.rpm` → `sudo rpm -i tailcat*.rpm` |
| Windows | `.zip` |

amd64 / arm64 / armv7 모두 제공.

### 방법 3: 소스에서 직접 빌드

**Go 1.27 이상** 필요 (https://go.dev/dl)

```bash
git clone https://github.com/bmshin94/tailcat
cd tailcat
go build -o tailcat ./cmd/tailcat
```

용량을 줄이려면 (공식 배포판과 동일한 방식, 약 16% 감소):

```bash
go build -tags "$(cat build-tags.txt)" -ldflags "-s -w" -o tailcat ./cmd/tailcat
```

클론 없이 한 번에 설치:

```bash
go install github.com/tailscale/tailcat/cmd/tailcat@latest
```

### 방법 4: Docker

```bash
docker pull ghcr.io/tailscale/tailcat:latest
docker run --rm -it ghcr.io/tailscale/tailcat:latest
```

### 방법 5: Nix / Arch

```bash
nix run github:tailscale/tailcat          # Nix
nix profile install github:tailscale/tailcat
yay -S tailcat-bin                        # Arch AUR (바이너리)
yay -S tailcat                            # Arch AUR (소스 빌드)
```

### 설치 확인

```bash
tailcat version
```

---

## 4. 사용법

### 기본 개념

```
┌─────────────┐                    ┌─────────────┐
│   서버 쪽    │  ─── 토큰 전달 ──→  │ 클라이언트 쪽 │
│  (기다리는)  │                    │  (접속하는)  │
└─────────────┘                    └─────────────┘
```

1. **서버 쪽**이 먼저 실행 → `tc...` 토큰 출력
2. 그 토큰을 **클라이언트 쪽**에 전달 → 접속

### ① 텍스트 주고받기 (가장 기본)

**받는 쪽:**
```bash
$ tailcat
# Selected bootstrap relay region 302, San Francisco
# 🐈 Server listening with new address: tcomFwWCCcjS5nKNqAod034nWo...
(대기)
```

**보내는 쪽:**
```bash
$ echo "안녕!" | tailcat tcomFwWCCcjS5nKNqAod034nWo...
```

받는 쪽 화면에 `안녕!` 이 출력됩니다.

### ② 파일 받기 — 드롭박스 모드 (추천)

**받는 쪽:**
```bash
$ tailcat recv ~/inbox
# 🐈 tcXXXX...
```

**보내는 쪽:**
```bash
$ tailcat cp 보고서.pdf tcXXXX...:
$ tailcat cp -r ./사진들 tcXXXX...:사진들      # 폴더 통째로
```

> 🔒 `recv`는 **쓰기 전용(write-only)** 입니다. 보내는 사람은 디렉터리 목록을
> 볼 수도, 기존 파일을 읽거나 덮어쓸 수도 없습니다.
>
> 실제 복사는 시스템 `scp`가 담당하므로 **진행률 표시**가 그대로 나옵니다.

### ③ 파일 나눠주기 (반대 방향)

**주는 쪽:**
```bash
$ tailcat serve files                    # 현재 폴더, 읽기 전용
$ tailcat serve --files=/pub:rw files    # 지정 폴더, 읽기+쓰기
$ tailcat serve --files=/inbox:wo files  # 쓰기 전용 드롭박스
```

**받는 쪽:**
```bash
$ tailcat ls -l tcXXXX...                # 목록 보기
$ tailcat cp tcXXXX...:보고서.pdf .        # 다운로드
```

| 접미사 | 의미 |
| --- | --- |
| `:ro` | 읽기 전용 (기본값) |
| `:rw` | 읽기 + 쓰기 |
| `:wo` | 쓰기 전용 (드롭박스) |

> 🛡️ 서버는 Go의 `os.Root`로 경로를 가두므로 `..` 나 심볼릭 링크로
> 지정 폴더 밖을 벗어날 수 없습니다.
>
> 프로토콜이 SFTP라서 일반 `sftp` / `scp` 클라이언트도 사용 가능하고,
> `tailcat ls` 는 SFTP를 자체 구현해서 OpenSSH가 없어도 동작합니다.

### ④ 로컬 포트 공개 (개발자용)

```bash
$ tailcat serve 8080              # 단일 포트
$ tailcat serve 8080,8443         # 여러 포트
$ tailcat serve 8000-8999         # 포트 범위
$ tailcat serve all               # 전체 (주의)
```

**클라이언트:**
```bash
$ tailcat tcXXXX... 8080
$ tailcat socks tcXXXX... curl http://server.tailcat:8080/
```

### ⑤ 원격 SSH 접속

**서버 쪽 (Linux/macOS):**
```bash
$ tailcat serve no-auth-ssh
```

**클라이언트 쪽:**
```bash
$ tailcat ssh tcXXXX...
$ tailcat ssh tcXXXX... ls -la          # 명령 하나만 실행
```

> ⚠️ `no-auth-ssh`는 **비밀번호가 없습니다.** 토큰을 아는 사람은 누구나
> 접속할 수 있으므로 반드시 `--allow` 와 함께 사용하세요.
> 비밀번호 인증을 원하면 `tailcat serve 22` 로 시스템 SSH에 연결하면 됩니다.

### ⑥ 연결 상태 확인

```bash
$ tailcat ping tcXXXX...
pong in 42.1ms via DERP(sfo)             # 중계 경유 (느림)
pong in 1.2ms via 203.0.113.7:41641      # 직통 (빠름)

$ tailcat ping --until-direct tcXXXX...  # 직통될 때까지 재시도
```

### ⑦ 기타 유용한 명령어

```bash
$ tailcat readme            # 전체 문서 출력
$ tailcat help              # 도움말
$ tailcat serve --help      # 하위 명령 도움말
$ tailcat parse tcXXXX...   # 토큰 내용을 JSON으로 출력
$ tailcat resolve tcXXXX... # 토큰에 DERP 정보를 embed (빠른 접속용)
$ tailcat printpub          # 내 클라이언트 공개키 출력
$ tailcat serve exit-node   # 서버의 네트워크 전체를 클라이언트에 개방
```

---

## 5. 열쇠(Key) 관리

### 일회용 키 (기본값)

`tailcat` 을 그냥 실행하면 매번 새 키·새 주소가 생성됩니다.
프로세스가 종료되면 그 주소는 **영구히 사라집니다.** → 안전한 기본값

```
# 🐈 Server listening with new address: tc...       ← "new address"
```

### 저장 키 (주소 고정)

```bash
$ tailcat genkey --key=default --region=nyc
# ~/.config/tailcat/keys/default.private.json 에 저장
```

이후에는 자동으로 사용됩니다:

```bash
$ tailcat serve 8080
# 🐈 Server listening with saved key "default": tc...   ← "saved key"
```

> 🎩 `default` 는 **매직 네임**입니다. 서버 모드가 자동으로 불러옵니다.
> (클라이언트 모드는 `client-default`)
>
> ⚠️ 단점: 예전에 그 주소를 알려준 사람은 **이후에도 계속** 접속할 수 있습니다.
> → 아래 `--allow` 필요

```bash
$ tailcat serve --key=new 8080              # 일회용으로 강제
$ tailcat genkey --list                     # 저장된 키 목록
$ tailcat genkey --delete --key=default     # 삭제
```

### 특정 클라이언트만 허용 (가장 안전한 설정)

**클라이언트 쪽 — 신원 키 생성:**
```bash
$ tailcat genkey --client --key=client-default
nodekey:cfb6bfa77a0654d7450947fd6acef17d2cd848da1d30b2540b13dac272ddfd16
```

**서버 쪽 — 그 클라이언트만 허용:**
```bash
$ tailcat serve --allow=nodekey:cfb6bf...ddfd16 22
```

> 🥷 허용되지 않은 클라이언트의 핸드셰이크는 **조용히 무시**됩니다.
> 서버가 존재한다는 사실조차 알 수 없습니다.

### DNS 이름으로 접속

TXT 레코드를 등록하면 토큰 대신 도메인을 쓸 수 있습니다.

```
my-server.example.com.  300  IN  TXT  "tailcat=tcXXXX..."
```

```bash
$ tailcat ssh my-server.example.com
$ tailcat ping my-server.example.com
$ tailcat my-server.example.com 8080
```

이 경우 지역을 고정해두는 것이 좋습니다:

```bash
$ tailcat genkey --key=default --fixed-region
```

`--fixed-region` 은 가장 가까운 DERP 지역을 **지금 한 번** 탐색해서
키와 토큰에 박아둡니다. 서버를 재시작해도 같은 지역에 붙으므로
DNS에 게시한 토큰이 계속 유효합니다.

---

## 6. 동작 원리

### 연결 토큰 (ConnBlob)

`tc` 접두사 + base64 인코딩된 [CBOR](https://cbor.io/) 데이터:

- 서버의 WireGuard 공개키 (Curve25519, 32바이트)
- 별도의 경로 탐색(disco) 공개키 (Curve25519, 32바이트)
- DERP 정보 — 지역 ID 정수 하나, 또는 DERP 서버 메타데이터 전체

지역 ID만 담으면 약 95바이트. `--full-address` 나 `tailcat resolve` 를 쓰면
DERP 정보가 embed된 긴 형태(자립형)가 됩니다.

### 네트워크 스택

| 구성요소 | 역할 |
| --- | --- |
| **WireGuard** | 유저스페이스 구현. TUN/TAP 장치 불필요 → 루트 권한 불필요 |
| **magicsock** | 직통 UDP와 DERP 릴레이를 다중화. STUN 기반 엔드포인트 탐색 |
| **Netstack (gVisor)** | 유저스페이스 TCP/IP 스택. OS 네트워크 설정 없이 연결 수락/발신 |
| **DERP 릴레이** | 랑데부 채널 + 직통 실패 시 폴백 경로 |

### 연결 흐름

1. **서버 시작** — 키 생성/로드 → DERP 릴레이 접속 → 토큰을 stderr에 출력 → 대기
2. **클라이언트가 토큰 파싱** — 서버 공개키·disco키·DERP 지역 파악 → 같은 릴레이 접속
3. **탐색 핸드셰이크** — 클라이언트가 `Meow` 메시지 전송 → 서버가 피어 목록에
   추가하고 `Meowed` 로 응답
4. **WireGuard 터널** — 양쪽이 피어로 설정됨 → 표준 WireGuard 핸드셰이크
   (초기엔 DERP 경유)
5. **NAT 통과** — 양쪽이 STUN으로 알아낸 공인 IP:포트와 로컬 주소를
   `call-me-maybe` 로 교환 → UDP 홀펀칭 시도 → 성공하면 **직통으로 승격**,
   실패해도 DERP 폴백으로 계속 동작
6. **데이터 전송** — 클라이언트가 터널 너머 TCP 포트로 접속 →
   서버가 포트별 핸들러로 분배 (localhost 포워딩 / stdout / SSH / SFTP 등)

---

## 7. PHP로 만들 수 있나요?

### ❌ tailcat 자체를 PHP로 재구현하는 것: 불가능

| tailcat이 하는 일 | PHP로 가능? |
| --- | --- |
| WireGuard 암호화 (Curve25519, ChaCha20) | 🟡 이론상 가능하지만 성능이 안 나옴 |
| UDP 홀펀칭 (STUN, 실시간 패킷 처리) | ❌ 저수준 실시간 처리에 부적합 |
| gVisor 유저스페이스 TCP/IP 스택 | ❌ 불가능 (Go/C 레벨 영역) |
| **장시간 연결 유지** | ❌ **결정적 문제** |

가장 큰 문제는 마지막 항목입니다. PHP-FPM의 실행 모델은
`요청 → 실행 → 응답 → 프로세스 종료` 인데,
tailcat은 **몇 시간~며칠 살아있어야** 합니다. 구조가 정반대입니다.

### ✅ PHP로 tailcat을 "조종하는" 서비스: 충분히 가능

```
┌──────────────────────────────┐
│  🖥️ PHP 웹 관리 시스템          │  ← 직접 만드는 부분 (제품의 가치)
│  로그인 / 기기목록 / 결제 /     │
│  접속버튼 / 로그 / 통계         │
└──────────────┬───────────────┘
               │ 프로세스 제어 & 토큰 교환
┌──────────────▼───────────────┐
│  🐈 tailcat 바이너리 (Go)       │  ← 갖다 쓰는 부분 (무료)
│  실제 암호화 터널 담당           │
└──────────────────────────────┘
```

### 연동 지점 3가지 (소스코드에서 확인)

#### ① `TAILCAT_ADDR_FILE` 환경변수 — `cmd/tailcat/tailcat.go:1261`

토큰을 파일에 쓰거나, `tcp:` 접두사를 붙이면 지정한 TCP 주소로 보내줍니다.

```php
<?php
// 드롭박스 서버를 띄우고 토큰 받아오기
$tokenFile = '/run/tailcat/' . bin2hex(random_bytes(8)) . '.addr';

$proc = proc_open(
    ['tailcat', 'serve', '--files=/srv/dropbox:wo', 'files'],
    [
        1 => ['file', '/var/log/tailcat.log', 'a'],
        2 => ['file', '/var/log/tailcat.err', 'a'],
    ],
    $pipes,
    null,
    ['TAILCAT_ADDR_FILE' => $tokenFile, 'HOME' => '/var/lib/tailcat']
);

// 토큰 파일이 생길 때까지 대기 (최대 10초)
for ($i = 0; $i < 100 && !is_file($tokenFile); $i++) {
    usleep(100_000);
}
$token = trim(file_get_contents($tokenFile));
// → DB에 저장하고 사용자에게 노출
```

#### ② `--json` 플래그 — `cmd/tailcat/tailcat.go:1259`

서버 모드에서 표준출력으로 `{"listenAddr":"tc..."}` 를 내보냅니다.

```php
<?php
$proc = proc_open(
    ['tailcat', 'serve', '--json', '8080'],
    [1 => ['pipe', 'w']],
    $pipes
);
$token = json_decode(fgets($pipes[1]), true)['listenAddr'];
```

#### ③ SOCKS5 프록시 — PHP curl에서 직접 사용 (가장 강력)

```bash
# 데몬으로 상시 실행 (systemd / supervisor)
$ tailcat socks --listen=127.0.0.1:1080 tcXXXX...
```

```php
<?php
// 원격 기기의 내부 웹서비스를 PHP에서 그대로 호출
$ch = curl_init("http://server.tailcat:8080/api/status");
curl_setopt($ch, CURLOPT_PROXY, '127.0.0.1:1080');
curl_setopt($ch, CURLOPT_PROXYTYPE, CURLPROXY_SOCKS5_HOSTNAME);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
$status = json_decode(curl_exec($ch), true);
```

> 💡 고객사 내부망 기기(공유기 뒤)를 **포트포워딩 없이** PHP에서 API 호출할 수 있습니다.
>
> ⚠️ tailcat 프로세스는 웹요청마다 띄우면 안 되고, `systemd` / `supervisor` 로
> **상시 데몬**으로 관리해야 합니다.

---

## 8. 수익화 아이디어

> ⚠️ 아래는 검증되지 않은 **가설**입니다. 실행 전 반드시 고객 인터뷰로 수요를 확인하세요.

### 🥇 1위: IoT / 키오스크 / 사이니지 원격관리 SaaS

- **문제:** 공장·매장 기기에 문제가 생기면 직접 방문해야 함
- **해결:** 기기에 tailcat 에이전트, PHP 대시보드에서 원격 접속
- **장점:** PHP 비중이 90% (대시보드/기기목록/알림/결제/로그), 고정 IP·포트포워딩·VPN 장비 불필요, B2B라 객단가 높음
- **수익모델:** 기기당 월 구독 + 초기 설치비

### 🥈 2위: 대용량 파일 납품 서비스 (B2B)

- **타겟:** 인쇄소, 영상제작사, 설계사무소, 사진스튜디오
- **문제:** 수 GB 파일을 웹하드로 주고받기 — 느리고 비싸고 보안 부담
- **해결:** PHP로 "납품 요청 링크" 발급 → tailcat으로 P2P 직송
- **장점:** **서버 스토리지·트래픽 비용이 사실상 0** (P2P라 서버를 안 거침) → 기존 웹하드와 원가 구조가 근본적으로 다름
- **수익모델:** 업체당 월 정액. "파일 크기 무제한"이 셀링포인트

### 🥉 3위: 중소기업용 원격지원 툴

- **문제:** 팀뷰어/애니데스크 라이선스 비용 부담
- **해결:** PHP 관리 페이지 + tailcat + 기존 RDP/VNC
  ```bash
  # 고객 PC
  tailcat serve 3389 --allow=nodekey:엔지니어공개키
  ```
- **장점:** `--allow` 로 지정된 엔지니어만 접속 (보안 어필), 병원·회계사무소 등 외부 클라우드 사용이 어려운 곳 타겟
- **확인 필요:** 화면 전송 성능은 실측 필요

### 4위: 구축·운영 대행 (가장 빠른 현금화)

- 소프트웨어가 아니라 **구축 서비스**를 판매
- tailcat + 자체 DERP 서버 세팅 → 구축비 + 유지보수 월 정액
- 개발 리스크 없이 즉시 시작 가능

> 💡 SaaS를 만들기 전에 이 방식으로 1~2건 수행하면서
> **실제 수요가 있는지 검증**하는 것이 가장 안전합니다.

### 권장 로드맵

```
1단계 (1주)  실제 PC 2대로 연결 테스트 → 속도/안정성 체감
2단계 (2주)  PHP 최소 데모: 버튼 클릭 → 토큰 발급 페이지
3단계 (1달)  잠재 고객 5명 인터뷰 → "돈 내고 쓸 의향?" 확인
4단계        수요 확인 후 → 자체 DERP 서버 + 본격 개발
```

**3단계를 건너뛰지 마세요.** 수요 검증 없이 장기 개발하는 것이 가장 큰 리스크입니다.

---

## 9. 주의사항

### 🔴 공용 DERP 릴레이로 유료 서비스를 하면 안 됩니다

README에 명시되어 있습니다:

> *"The public rate-limited Tailcat DERP relays have no uptime SLAs or
> throughput targets, and **we may revoke access to them at any time,
> for any reason.**"*

즉, 상용 서비스를 하려면 **자체 DERP 서버 운영이 필수**입니다.

```bash
$ tailcat genkey --key=default --region=derp.내도메인.com
```

이렇게 하면 클라이언트도 Tailscale의 DERP 맵 서버나 릴레이에 접속하지 않습니다.
DERP 서버는 TLS 인증서가 있는 도메인이 필요하며(derper가 Let's Encrypt로
자동 발급 가능), 서버 비용이 발생합니다. **원가 계산에 반드시 포함하세요.**

→ https://github.com/tailscale/tailscale/tree/main/cmd/derper#derp

여러 릴레이를 운영한다면 자체 DERP 맵 JSON을 서빙하고 양쪽에서
`--derpmap-url` 로 가리키면 됩니다.

### 🟡 라이선스 (BSD 3-Clause)

- ✅ 상업적 사용 가능
- ✅ 소스코드 공개 의무 없음 (GPL 아님)
- ⚠️ 저작권 고지문을 문서/배포물에 포함해야 함
- ❌ Tailscale의 이름으로 파생 제품을 홍보/보증하는 것 금지 (3번 조항)

### 🟡 API / CLI 안정성 보장 없음

> *"it comes with no API or CLI stability promises"*

Go API, CLI 플래그, 출력 형식, 와이어 포맷 모두 변경될 수 있습니다.
**버전을 고정**하고 업데이트 전 테스트하세요.

### 🟡 경쟁 환경

| 경쟁자 | 강점 |
| --- | --- |
| ngrok | 개발자 시장 선점 |
| Cloudflare Tunnel | 무료 + 강력함 |
| Tailscale 본체 | 원조, 넉넉한 무료 플랜 |

> 터널 기술 자체를 파는 것으로는 경쟁이 어렵습니다.
> **특정 업종의 특정 문제를 해결하는 관리 시스템**을 팔아야 합니다.
> (인쇄소가 원하는 건 "P2P 터널"이 아니라 "납품 파일 관리 시스템"입니다.)

### 🟡 기타 운영 주의사항

| 항목 | 내용 |
| --- | --- |
| 🎫 토큰 관리 | 토큰 = 비밀번호. 공개 저장소·채팅에 노출 금지 |
| 🐌 속도 | 공용 릴레이는 속도 제한. 직통 연결 시 빠름 |
| 📦 압축 | 전송 시 압축하지 않음 (SFTP·SSH 계층 모두). 큰 파일은 미리 압축 |
| 🔓 no-auth-ssh | 인증 없음. 반드시 `--allow` 와 병행 |
| 🌐 브라우저 데모 | DERP 릴레이 경유만 가능 (WebRTC 미지원) |

---

## 10. 검증 기록

이 문서를 작성하며 실제로 실행해 확인한 내용입니다.

### ✅ 성공한 것

| 항목 | 결과 |
| --- | --- |
| Go 툴체인 | `go1.27.0 linux/amd64` |
| 빌드 | `go build ./cmd/tailcat` — **49초, 30MB 바이너리 생성** |
| 버전 확인 | `tailcat version` → `v0.0.0-20260901025231-5e4e61440c77` |
| 도움말 | `tailcat help` 정상 출력 |
| 키 생성 | `tailcat genkey --key=demo --region=derp.example.com` 성공 |
| 토큰 파싱 | `tailcat parse` 성공 |

`tailcat parse` 실제 출력:

```json
{
    "ServerPublic": "nodekey:8be5192a1ef0bbfb76b77d94e70fb79fec341f2faf4e2beb3ce3a816e9b97a0c",
    "ServerDiscoPublic": "discokey:9901b40fefc39a8a40455749cde057356444d074156424b308a6ccca9f9c3326",
    "Region": [
        {
            "Nodes": [
                { "HostName": "derp.example.com" }
            ]
        }
    ]
}
```

### ❌ 확인하지 못한 것

**실제 P2P 연결 테스트는 하지 못했습니다.**

```
Expand: fetching DERPMap for region -1:
Get "https://tailcat.dev/derpmap.json": Forbidden
```

이는 tailcat의 문제가 아니라, 문서 작성에 사용한 **클라우드 샌드박스 환경의
네트워크 정책**이 외부 도메인을 차단했기 때문입니다
(프록시가 `tailcat.dev:443` CONNECT 요청에 403 응답).

일반 PC 환경에서는 정상 동작할 것으로 예상되나,
**직접 2대의 기기로 검증한 뒤 사업화를 진행하세요.**

### 첫 실습 (터미널 2개로 5분)

**터미널 1:**
```bash
cd tailcat
go build -o tailcat ./cmd/tailcat
./tailcat
# 출력되는 tc... 토큰 복사
```

**터미널 2:**
```bash
echo "hello tailcat" | ./tailcat <복사한_토큰>
```

터미널 1에 `hello tailcat` 이 출력되면 성공입니다.

---

## 부록: Go 라이브러리로 사용하기

CLI 없이 Go 프로그램에 직접 임베드할 수도 있습니다.

**서버:**
```go
package main

import (
	"fmt"
	"log"
	"net"

	"github.com/tailscale/tailcat"
)

func main() {
	s := &tailcat.Server{
		OnTCP: func(port uint16) func(net.Conn) {
			return func(c net.Conn) {
				fmt.Fprintf(c, "hello from port %v\n", port)
				c.Close()
			}
		},
	}
	if err := s.Start(); err != nil {
		log.Fatal(err)
	}
	fmt.Println(s.ConnBlob())
	select {}
}
```

**클라이언트:**
```go
package main

import (
	"context"
	"io"
	"log"
	"os"

	"github.com/tailscale/tailcat"
)

func main() {
	cl := tailcat.NewClient(tailcat.ConnBlob(os.Args[1]))
	defer cl.Close()
	c, err := cl.DialTCPPort(context.Background(), 80)
	if err != nil {
		log.Fatal(err)
	}
	io.Copy(os.Stdout, c)
}
```

`Server` / `Client` 모두 **제로값이 동작**합니다.
설정하지 않은 항목은 기본값(임시 키, 가장 가까운 DERP 지역, `log.Printf` 로거)이
자동 적용되며, 조용히 쓰려면 `Logf` 를 `logger.Discard` 로 설정하면 됩니다.

---

*이 문서는 https://github.com/bmshin94/tailcat 저장소를 분석하여 작성되었습니다.*
