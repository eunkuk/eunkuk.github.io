---
title: "DNS 포트(53) 방화벽 장애 트러블슈팅"
date: 2026-02-12 16:00:00 +0900
categories: [Dev, Infra]
tags: [dns, firewall, troubleshooting, network, idc]
description: "IDC 환경에서 DNS 포트(UDP/TCP 53) 아웃바운드 차단으로 인한 도메인 resolve 실패를 추적하고 해결한 경험을 공유한다."
---

## 배경

입사 첫날, IDC에 있는 서버 몇 대를 넘겨받았다. 인프라 구성도, 네트워크 토폴로지도, 방화벽 정책 문서도 없었다. 이전 담당자는 이미 퇴사한 상태였고 인수인계는 사실상 없었다.

> "서버 접속 정보는 메일로 보내드렸습니다. 나머지는 알아서 파악해주세요."

그렇게 받은 건 서버 IP, SSH 접속 계정 정보가 전부였다.

## 증상 발견

서버에서 외부 API를 호출하는 애플리케이션을 올리는 과정에서 문제가 시작됐다.

```bash
# 소켓으로 IP 직접 통신 → 정상
$ curl http://203.0.113.10/api/health
{"status": "ok"}

# 도메인 기반 통신 → 실패
$ curl http://api.example.com/health
curl: (6) Could not resolve host: api.example.com
```

IP를 직접 넣으면 정상 응답이 오는데, 도메인으로 요청하면 resolve 자체가 안 됐다. 처음에는 애플리케이션 설정 문제인 줄 알았다.

## 원인 추적

### 1단계: DNS resolve 확인

가장 먼저 `nslookup`으로 DNS 동작을 확인했다.

```bash
$ nslookup api.example.com
;; connection timed out; no servers could be reached
```

DNS 서버에 아예 연결이 되지 않았다. `/etc/resolv.conf`{: .filepath}를 확인해봤다.

```bash
$ cat /etc/resolv.conf
nameserver 8.8.8.8
nameserver 8.8.4.4
```

설정 자체는 문제가 없어 보였다. Google Public DNS를 사용하고 있었다.

### 2단계: IP 기반 통신 검증

혹시 네트워크 전체가 막힌 건 아닌지 확인했다.

```bash
# ping → 정상
$ ping -c 4 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=3.42 ms
...

# 외부 웹 서버에 직접 telnet → 정상
$ telnet 203.0.113.10 80
Trying 203.0.113.10...
Connected to 203.0.113.10.
```

IP 기반 통신은 정상이었다. **ICMP도 되고 TCP도 되는데 DNS만 안 된다.** 여기서 문제 범위가 확 좁혀졌다.

### 3단계: DNS 포트 직접 테스트

DNS는 UDP 53번 포트를 사용한다. 해당 포트로 직접 통신을 시도했다.

```bash
# DNS 포트(53) 직접 연결 시도
$ telnet 8.8.8.8 53
Trying 8.8.8.8...

# 응답 없음 — 타임아웃
```

`dig`로도 확인했다.

```bash
$ dig @8.8.8.8 api.example.com
;; connection timed out; no servers could be reached
```

DNS 서버의 IP(8.8.8.8)에는 ping이 되지만, 53번 포트로의 통신은 완전히 막혀 있었다.

### 문제 범위 정리

| 테스트 | 결과 | 의미 |
|--------|------|------|
| `ping 8.8.8.8` | 성공 | ICMP 아웃바운드 허용 |
| `telnet 203.0.113.10 80` | 성공 | TCP 80 아웃바운드 허용 |
| `telnet 8.8.8.8 53` | 실패 | TCP 53 아웃바운드 차단 |
| `nslookup api.example.com` | 실패 | UDP 53 아웃바운드 차단 |

## 근본 원인

**IDC 방화벽에서 DNS 포트(UDP/TCP 53) 아웃바운드가 차단되어 있었다.**

이전 담당자가 방화벽을 설정할 때 필요한 포트만 열고 나머지는 전부 막아놓은 것으로 보였다. HTTP(80), HTTPS(443), ICMP 정도만 허용된 상태였고, DNS 포트는 빠져 있었다.

IDC 내부에 별도의 DNS 서버가 없었기 때문에 외부 DNS 서버(8.8.8.8)로의 쿼리가 필수였지만, 그 경로가 막혀 있던 것이다.

> 방화벽 화이트리스트 방식에서 DNS 포트를 누락하면 IP 기반 통신은 정상이지만 모든 도메인 기반 통신이 실패한다.
{: .prompt-warning }

## 해결

IDC 방화벽 관리 담당자에게 요청하여 아웃바운드 규칙에 DNS 포트를 추가했다.

```
# 추가된 방화벽 규칙
출발지: 서버 IP 대역
목적지: 8.8.8.8, 8.8.4.4
프로토콜/포트: UDP 53, TCP 53
방향: Outbound
동작: Allow
```

정책 적용 후 즉시 정상 동작을 확인했다.

```bash
$ nslookup api.example.com
Server:    8.8.8.8
Address:   8.8.8.8#53

Non-authoritative answer:
Name:   api.example.com
Address: 203.0.113.10

$ curl http://api.example.com/health
{"status": "ok"}
```

## 교훈

### 인수인계의 중요성

문서가 없는 인프라는 블랙박스다. 서버 접속 정보만으로는 네트워크 구성을 파악할 수 없다. 최소한 아래 항목은 문서화되어 있어야 한다.

- 네트워크 토폴로지 (서브넷, VLAN, 게이트웨이)
- 방화벽 정책 (인바운드/아웃바운드 규칙 목록)
- DNS 구성 (내부 DNS 서버 유무, 사용 중인 DNS)
- 서버 역할별 필요 포트 목록

### 네트워크 트러블슈팅 체계적 접근법

이번 경험을 통해 정리한 네트워크 장애 추적 순서다.

1. **증상 정의** — 정확히 무엇이 되고 무엇이 안 되는지 구분
2. **계층별 테스트** — ICMP(ping) → TCP(telnet) → 애플리케이션(curl) 순으로 확인
3. **변수 분리** — IP vs 도메인, 포트별, 프로토콜별로 하나씩 바꿔가며 테스트
4. **문제 범위 축소** — 테스트 결과를 표로 정리하면 패턴이 보인다
5. **근본 원인 확인** — 추측이 아닌 직접 확인 (방화벽 규칙 조회, 패킷 캡처 등)

> "될 때와 안 될 때의 차이를 찾으면 원인이 보인다."
{: .prompt-tip }
