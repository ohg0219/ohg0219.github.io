---
title: "Docker 네트워크 [2편]"
description: "bridge, host, custom"
date: 2026-05-12 16:20:00 +0900
categories: [infra]
tags: [docker, network, bridge, port-mapping, container-dns]
image:
  path: /assets/img/posts/docker-network/cover.jpg
  alt: Docker 컨테이너들이 브리지 네트워크와 포트 매핑으로 연결되는 구조를 표현한 기술 커버 이미지
mermaid: true
series:
  name: docker
  part: 2
---

{% include series-nav.html %}

> 컨테이너 두 개를 띄우자마자 마주치는 질문은 똑같습니다. **"얘들 어떻게 서로 부르지?"**

웹 서버와 DB 컨테이너를 띄우긴 했는데, 웹 서버가 DB 에 접속하려면 호스트를 어떻게 적어야 할까요. `localhost` 는 통하지 않습니다. 컨테이너 입장에서 `localhost` 는 자기 자신입니다. 호스트 IP 를 직접 적자니 컨테이너가 옮겨가거나 재시작될 때마다 불안합니다.

Docker 가 이 문제를 다루는 방식이 **네트워크 드라이버** 와 **DNS** 입니다.

이번 편은 기본 네트워크 세 종류, port mapping 이 실제로 무엇을 하는지, 컨테이너 이름을 DNS 처럼 쓰는 방법, 그리고 운영에서 자주 마주치는 네트워크 함정을 정리합니다.

## 기본 네트워크 세 종류

Docker 는 설치 직후 세 가지 네트워크를 자동으로 만들어 둡니다.

```powershell
docker network ls
```

```
NETWORK ID     NAME      DRIVER    SCOPE
abc123...      bridge    bridge    local
def456...      host      host      local
ghi789...      none      null      local
```

| 드라이버 | 동작 | 쓰임 |
|---|---|---|
| **bridge** | Docker 가 만든 가상 네트워크. 컨테이너끼리는 같은 bridge 안에서 IP 로 통신 가능. 외부에서 접근하려면 port mapping 필요 | **기본값.** 옵션 안 주면 모든 컨테이너가 여기 들어감 |
| **host** | 컨테이너가 호스트의 네트워크 스택을 그대로 사용. 격리 없음. port mapping 불필요 | 성능이 critical 하거나 호스트 IP 가 필요한 시스템 도구. Linux 에서만 정상 동작 (macOS·Windows 는 VM 안 호스트라 의도와 다르게 동작) |
| **none** | 네트워크 인터페이스 자체가 없음 | 네트워크가 필요 없는 일회성 작업, 격리 컨테이너 |

가장 자주 쓰는 건 단연 **bridge** 입니다. 대부분의 경우 이걸로 충분합니다. 다만 **기본 bridge 와 사용자가 만든 custom bridge 는 다르다** 는 점이 곧 핵심으로 나옵니다.

## port mapping 이 실제로 하는 일

`-p 8080:80` 옵션을 무심코 써왔을 텐데, 이 한 줄이 안에서 하는 일은 다음과 같습니다.

```mermaid
flowchart LR
    Client[브라우저<br/>localhost:8080] --> Host[호스트<br/>8080 포트]
    Host --> NAT[Docker iptables<br/>NAT 규칙]
    NAT --> Container[컨테이너<br/>80 포트 / nginx]
```

호스트의 8080 으로 들어오는 트래픽을 NAT 규칙으로 컨테이너의 80 으로 forward 합니다. 컨테이너 자체는 자기 80 포트만 신경 쓰면 됩니다.

자주 쓰는 변종:

- `-p 8080:80` — 호스트 8080 → 컨테이너 80
- `-p 127.0.0.1:8080:80` — 호스트의 loopback 만 받음 (외부 접속 차단)
- `-p 80` — 호스트의 임의 포트를 자동 할당 (`docker ps` 로 확인)
- `-P` (대문자) — Dockerfile 의 `EXPOSE` 에 명시된 모든 포트를 자동 할당

운영에서는 외부 노출이 필요한 경우만 명시적으로 `-p` 를 주는 게 안전합니다. 내부 통신은 다음에 나오는 custom network 로 해결합니다.

## 컨테이너 이름을 DNS 처럼 쓰기

기본 `bridge` 네트워크에는 **DNS 가 동작하지 않습니다.** 컨테이너 이름으로 `ping nginx` 가 안 됩니다. IP 로만 접근 가능한데, 컨테이너 IP 는 재시작 때마다 바뀌므로 실사용에는 적합하지 않습니다.

해결책은 **custom network** 를 만드는 것입니다.

```powershell
docker network create my-app

docker run -d --name db  --network my-app postgres:16
docker run -d --name web --network my-app -p 8080:80 my-web-image
```

같은 custom network 에 두 컨테이너를 두면, 컨테이너 안에서:

```powershell
docker exec -it web ping db
docker exec -it web curl http://db:5432
```

`db` 라는 컨테이너 이름이 그대로 호스트네임처럼 동작합니다. DB 컨테이너가 재시작되어 IP 가 바뀌어도 이름은 그대로라 연결이 깨지지 않습니다.

이게 마이크로서비스 / docker-compose 가 동작하는 메커니즘입니다. compose 가 알아서 custom network 를 만들어 주기 때문에 서비스끼리 이름으로 부를 수 있습니다.

> 정리: **컨테이너 이름으로 통신하고 싶으면 custom network 부터 만든다.** 기본 bridge 에 띄우고 "왜 안 되지" 헤매는 게 가장 자주 보는 실수.

## 자주 마주치는 세 장면

**1. "컨테이너 안에서 호스트로 접근하고 싶다"**

컨테이너에서 호스트 PC 에 떠 있는 DB 에 붙고 싶을 때. `localhost` 는 컨테이너 자기 자신을 가리키므로 안 됩니다.

- macOS · Windows: `host.docker.internal` 이라는 특수 호스트네임이 호스트를 가리킨다
- Linux: `--add-host=host.docker.internal:host-gateway` 옵션을 명시적으로 줘야 동일하게 동작한다

**2. "두 컨테이너가 서로 통신이 안 된다"**

대부분은 **같은 네트워크에 있지 않음**. `docker network inspect <network>` 로 어떤 컨테이너가 들어 있는지 확인합니다.

```powershell
docker network inspect my-app
```

출력의 `Containers` 섹션에 두 컨테이너가 모두 보이는지 체크. 없으면 `docker network connect my-app <container>` 로 붙입니다.

**3. "포트가 이미 사용 중이라며 컨테이너가 안 뜬다"**

`Error response from daemon: ... bind: address already in use.` 호스트의 해당 포트를 다른 프로세스가 쓰고 있다는 뜻입니다. Windows 라면 PowerShell 에서:

```powershell
netstat -ano | findstr :8080
```

해당 PID 를 잡아 종료하거나, `-p` 의 호스트 포트를 다른 번호로 바꿉니다.

## 가져갈 한 줄

> **컨테이너 통신은 "같은 네트워크에 있는가" 하나만 보면 된다.**

격리·DNS·port mapping 같은 개념들은 결국 *"누가 누구를 볼 수 있는가"* 를 다루는 도구입니다. 트러블슈팅도 같습니다. 우선 `docker network inspect` 로 같은 네트워크 안에 있는지부터 보면 절반은 풀립니다.

다음 편은 이 모든 걸 한 줄로 묶는 **Docker Compose** 입니다. 여러 컨테이너를 YAML 하나로 선언하고, custom network 도 알아서 만들어주는 도구를 다룹니다.

---

## 참고

- [Docker Networking — overview](https://docs.docker.com/network/)
- [Docker Networking — drivers](https://docs.docker.com/network/drivers/)
