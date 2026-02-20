---
title: "소켓은 되는데 왜 Google Chat API만 안 되는 거야 — DNS 포트 차단 트러블슈팅"
date: 2025-11-05 22:00:00 +0900
categories: [Dev, Infra]
tags: [dns, firewall, troubleshooting, network, idc, retrospective]
description: "IDC 환경에서 소켓 통신은 정상인데 에러 모니터링과 Google Chat API만 실패하는 현상의 원인을 추적한 기록."
---

## 상황

IDC에 있는 서버 몇 대를 넘겨받았다. 인프라 구성도, 네트워크 토폴로지도, 방화벽 정책 문서도 없었다. 이전 담당자는 이미 퇴사한 상태였고 인수인계는 사실상 없었다.

> "서버 접속 정보는 메일로 보내드렸습니다. 나머지는 알아서 파악해주세요."

받은 건 서버 IP, SSH 접속 계정 정보가 전부였다.

그래도 서비스 자체는 돌아가고 있었다. 소켓 기반으로 클라이언트와 통신하는 구조였는데, 접속도 잘 되고 데이터도 정상적으로 오갔다. 서버 상태만 틈틈이 확인하면 되겠거니 했다.

문제는 에러 모니터링을 붙이려고 하면서 시작됐다.

## 증상

장애 발생 시 Google Chat으로 알림을 보내는 기능을 추가하려고 했다. 에러가 터지면 Webhook으로 Google Chat API를 호출해서 팀 채팅방에 메시지를 쏘는, 흔한 구성이다.

로컬에서는 잘 되는 걸 확인하고 서버에 올렸는데, 알림이 안 왔다. 일부러 에러를 발생시켜도 조용했다.

```bash
# 서버에서 직접 Google Chat Webhook 호출
$ curl -X POST "https://chat.googleapis.com/v1/spaces/..." \
  -H "Content-Type: application/json" \
  -d '{"text": "테스트 알림"}'
curl: (28) Connection timed out after 30000 milliseconds
```

30초 기다리다가 타임아웃. 처음에는 Google 서버가 응답을 안 하는 건가 싶었다.

근데 이상한 점은, **기존 소켓 통신은 아무 문제 없이 돌아가고 있었다**는 것이다. 서버가 네트워크에서 격리된 것도 아닌데 왜 Google Chat API만 안 되지?

처음에는 단순하게 생각했다. Google 쪽 URL이 잘못됐나, 방화벽에서 Google IP 대역을 막고 있나, 아니면 HTTPS만 문제인가. 이것저것 시도해봤는데 다 아니었다.

## 삽질

### "HTTPS 문제인가?"

혹시 TLS 관련 이슈인가 싶어서 HTTP로도 테스트해봤다. 아무 외부 도메인이나 넣어봤다.

```bash
$ curl http://httpbin.org/get
curl: (28) Connection timed out after 30000 milliseconds
```

HTTPS 문제가 아니었다. **도메인이 들어가는 요청은 전부 타임아웃이 떨어졌다.** IP를 직접 넣으면 바로 응답이 오는데, 도메인만 들어가면 30초 멈춰있다가 죽는다.

### "DNS 설정이 잘못됐나?"

```bash
$ cat /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
```
{: file="/etc/resolv.conf" }

Google Public DNS. 설정 자체는 멀쩡했다.

### "네트워크가 아예 막힌 건가?"

외부로 아예 나가지 못하는 건가 싶어서 ping을 쳐봤다.

```bash
$ ping -c 4 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

--- 8.8.8.8 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3003ms
```

ping도 안 된다. 이쯤 되면 "이 서버 외부 통신 자체가 안 되는 거 아냐?" 싶었다.

근데 기존 소켓 통신은 잘 되고 있었고, IP 직접 호출도 됐다.

```bash
$ curl http://203.0.113.10/api/health
{"status": "ok"}
```

뭔가 되는 것도 있고 안 되는 것도 있는, 애매한 상황이었다.

### "서버 자체에서 막고 있나?"

혹시 서버 로컬 방화벽이 문제인가 싶어서 iptables를 확인했다.

```bash
$ sudo cat /etc/sysconfig/iptables
```

```text
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT

# -- 이하 회사/IDC 관련 허용 IP 규칙 다수 --

#ALL ANY RUL
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8888 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 3306 -j ACCEPT

-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A INPUT -p icmp -m icmp --icmp-type 8 -j DROP
-A INPUT -j DROP
COMMIT
```
{: file="/etc/sysconfig/iptables" }

일단 OUTPUT 체인은 ACCEPT라서 **서버에서 나가는 트래픽을 막고 있지는 않았다.** 아웃바운드 문제는 서버 iptables가 아니라 IDC 네트워크 단이라는 걸 확인했다.

그런데 이걸 보면서 다른 게 눈에 밟혔다. 기본 정책이 `INPUT ACCEPT`인데 맨 아래에서 `REJECT`랑 `DROP`으로 막고 있다. 특정 IP에 대해서는 `-j ACCEPT`로 모든 포트를 열어주고 있고, 주석도 `#ALL ANY RUL`처럼 중간에 잘려있고, 테스트용 규칙도 그대로 남아있다. 누가 언제 왜 추가한 건지 알 수 없는 규칙들이 쌓여있었다.

DNS 문제를 해결하고 나서, iptables도 정리하기로 했다. 기본 정책을 `DROP`으로 바꾸고, 쓰지 않는 규칙은 주석 처리하고, 구조를 깔끔하게 다시 잡았다.

```text
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT

# -- 이하 회사별 허용 IP (사용 중인 것만 남김) --

-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8888 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 3306 -j ACCEPT
COMMIT
```
{: file="/etc/sysconfig/iptables (정리 후)" }

기본 정책이 `DROP`이니까 맨 아래에 `REJECT`나 `DROP`을 따로 쓸 필요가 없다. 허용 규칙에 매칭되지 않으면 자동으로 버려진다. 이전 설정은 "일단 다 열어놓고 맨 아래에서 막는" 구조였는데, 이걸 "기본적으로 다 막고 필요한 것만 여는" 구조로 바꾼 거다.

> iptables 기본 정책이 ACCEPT인 상태에서 맨 아래 DROP으로 막는 방식은, 규칙 순서가 꼬이거나 실수로 DROP 위에 ACCEPT가 추가되면 의도치 않게 열릴 수 있다. 기본 정책을 DROP으로 두는 게 더 안전하다.

### 정리하고 나니 보였다

머리가 꼬이길래 되는 것과 안 되는 것을 정리해봤다.

| 테스트 | 결과 |
|--------|------|
| 소켓 통신 (IP 직접 연결) | ✅ 정상 |
| `ping 8.8.8.8` | ❌ 100% packet loss |
| `curl http://203.0.113.10/...` (IP 직접) | ✅ 정상 |
| `curl https://chat.googleapis.com/...` (도메인) | ❌ 타임아웃 |
| `nslookup api.example.com` | ❌ 타임아웃 |

**IP + TCP로 때리면 되는데, ping(ICMP)도 안 되고 도메인도 안 된다.** 허용된 포트(80, 443, 소켓 포트)만 열려있고 나머지는 전부 막혀있는 구조라는 느낌이 왔다.

그제서야 "이건 DNS 자체의 문제가 아니라, DNS 쿼리를 보내는 경로가 막혀 있는 거 아닌가?" 하는 생각이 들었다. 허용된 TCP 포트로만 통신이 되고, 그 외에는 ICMP든 UDP든 전부 차단되어 있는 구조.

## 원인 확인

DNS는 UDP 53번 포트를 사용한다. 해당 포트로 직접 통신을 시도했다.

```bash
$ telnet 8.8.8.8 53
Trying 8.8.8.8...
# 응답 없음 — 타임아웃
```

```bash
$ dig @8.8.8.8 chat.googleapis.com
;; connection timed out; no servers could be reached
```

8.8.8.8에 ping도 안 되고, 53번 포트로도 통신이 안 됐다. **방화벽에서 허용된 TCP 포트 외에는 전부 차단하고 있었다.** DNS 포트(UDP/TCP 53)도 당연히 막혀 있었고.

기존 소켓 통신이 잘 된 이유도 설명이 됐다. 소켓은 IP 주소로 직접 연결하는 구조였기 때문에 DNS가 필요 없었던 것이다. 이전 담당자가 방화벽을 설정할 때 서비스에 필요한 포트만 열고 나머지는 전부 막아놓은 거였다. HTTP(80), HTTPS(443), 소켓 통신 포트 정도만 허용된 상태였고, ICMP도, DNS 포트도 빠져 있었다.

서비스 운영에는 문제가 없으니 아무도 몰랐고, 새로운 외부 API를 도메인으로 호출하려는 순간 비로소 드러난 거다.

> 방화벽을 화이트리스트 방식으로 운영하면서 DNS 포트를 누락하면, 허용된 TCP 포트로의 IP 직접 통신은 정상이지만 도메인 기반 통신은 전부 실패한다. 기존 서비스가 IP로만 통신하고 있었다면 이 문제는 영원히 안 터질 수도 있다.

## 해결

IDC 방화벽 관리 담당자에게 요청하여 아웃바운드 규칙에 DNS 포트를 추가했다.

| 항목 | 값 |
|------|-----|
| 출발지 | 서버 IP 대역 |
| 목적지 | 8.8.8.8, 8.8.4.4 |
| 프로토콜/포트 | UDP 53, TCP 53 |
| 방향 | Outbound |
| 동작 | Allow |

적용 후 확인:

```bash
$ nslookup chat.googleapis.com
Server:    8.8.8.8
Address:   8.8.8.8#53

Non-authoritative answer:
Name:   chat.googleapis.com
Address: 142.250.196.78

$ curl -X POST "https://chat.googleapis.com/v1/spaces/..." \
  -H "Content-Type: application/json" \
  -d '{"text": "테스트 알림"}'
# 200 OK — 알림 수신 확인
```

방화벽 규칙 한 줄이 원인이었다. 원인을 찾고 나면 항상 허무한데, 찾기까지의 과정이 문제다.

## 돌아보며

이번 건은 기술적으로 어려운 문제는 아니었다. DNS 포트가 막혀 있었다, 그게 전부다.

그런데 "소켓은 되는데 왜 이건 안 되지?"라는 상황에서 DNS 포트 차단까지 도달하는 데 생각보다 시간이 걸렸다. 기존 서비스가 잘 돌아가고 있으니 네트워크는 문제없을 거라는 전제가 머릿속에 깔려 있었고, 그 전제가 사고 범위를 좁혔다.

결국 도움이 된 건 **되는 것과 안 되는 것을 표로 정리한 것**이었다. 머릿속으로만 돌리면 "이것도 되는데 왜 저건 안 되지" 하면서 빙글빙글 돌기만 하는데, 적어놓으면 패턴이 보인다. IP는 되고 도메인은 안 된다 — 여기까지 정리되면 DNS 문제라는 건 자명하다.

### 네트워크 트러블슈팅 순서

이번 경험을 통해 정리한 접근 순서다.

> 1. **증상 정의** — 정확히 무엇이 되고 무엇이 안 되는지 구분한다
> 2. **계층별 테스트** — ICMP(ping) → TCP(telnet) → 애플리케이션(curl) 순으로 확인
> 3. **변수 분리** — IP vs 도메인, 포트별, 프로토콜별로 하나씩 바꿔가며 테스트
> 4. **표로 정리** — 테스트 결과를 적어놓으면 패턴이 보인다
> 5. **근본 원인 확인** — 추측이 아닌 직접 확인 (방화벽 규칙 조회, 패킷 캡처 등)

### 인수인계에 대해

문서가 없는 인프라는 블랙박스다. 최소한 이 정도는 남겨져 있어야 한다.

- 네트워크 토폴로지 (서브넷, VLAN, 게이트웨이)
- 방화벽 정책 (인바운드/아웃바운드 규칙 목록)
- DNS 구성 (내부 DNS 서버 유무, 사용 중인 DNS)
- 서버 역할별 필요 포트 목록

다음 사람이 또 삽질하지 않게, 나라도 정리해두자는 생각으로 이 글을 쓴다.
