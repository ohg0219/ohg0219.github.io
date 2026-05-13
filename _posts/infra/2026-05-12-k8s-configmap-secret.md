---
title: "ConfigMap 과 Secret [5편]"
description: "설정과 비밀을 이미지에서 분리"
date: 2026-05-12 21:33:00 +0900
categories: [infra]
tags: [kubernetes, k8s, configmap, secret, env, volume-mount]
image:
  path: /assets/img/posts/k8s-configmap-secret/cover.png
  alt: Kubernetes Pod 에 ConfigMap 과 Secret 이 설정과 비밀 값으로 주입되는 모습을 표현한 기술 커버 이미지
mermaid: true
series:
  name: kubernetes
  part: 5
---

{% include series-nav.html %}

> **DB 접속 정보를 이미지에 넣어서 빌드했는데, 스테이징과 운영이 같은 DB 를 보고 있었습니다.**

Docker 를 쓸 때도 마찬가지였습니다. 설정값을 이미지 안에 하드코딩하면, 환경이 바뀔 때마다 이미지를 다시 빌드해야 합니다. Docker 에서는 `-e` 옵션이나 `.env` 파일로 환경 변수를 주입했습니다.

K8s 에서는 이 역할을 **ConfigMap** 과 **Secret** 이 합니다.

이번 편은 설정과 비밀을 이미지에서 분리하는 두 가지 도구, 그리고 "Secret 이 정말 안전한가?" 에 대한 현실적인 기준을 정리합니다.

## 왜 설정을 이미지에서 분리해야 하나

같은 앱이지만 환경에 따라 달라지는 값이 있습니다:

- DB 주소 — 개발은 `localhost:5432`, 운영은 `rds.amazonaws.com:5432`
- 로그 레벨 — 개발은 `DEBUG`, 운영은 `WARN`
- API 키 — 환경마다 다름
- 기능 플래그 — 스테이징에서만 켜진 기능

이런 값을 이미지에 넣으면:

1. 환경마다 **이미지를 따로 빌드**해야 한다
2. 비밀(패스워드, API 키)이 **이미지 레이어에 남는다** — Docker 시리즈 5편에서 다뤘던 문제
3. 설정 하나 바꾸려고 이미지를 다시 빌드하고, 다시 배포해야 한다

원칙은 간단합니다: **이미지는 하나, 설정은 밖에서 주입한다.**

## ConfigMap — 일반 설정을 담는 통

**ConfigMap** 은 key-value 형태의 설정 데이터를 저장하는 K8s 리소스입니다. 비밀이 아닌 일반 설정값을 담습니다.

### ConfigMap 만들기

**방법 1 — 명령어로:**

```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=postgres-svc \
  --from-literal=LOG_LEVEL=INFO
```

**방법 2 — YAML 로:**

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "postgres-svc"
  LOG_LEVEL: "INFO"
  APP_MODE: "production"
```

```bash
kubectl apply -f configmap.yaml
```

**방법 3 — 파일 통째로:**

설정 파일 자체를 ConfigMap 에 넣을 수도 있습니다.

```bash
kubectl create configmap nginx-conf --from-file=nginx.conf
```

이러면 `nginx.conf` 파일 내용이 통째로 ConfigMap 에 저장됩니다.

### ConfigMap 확인

```bash
kubectl get configmap app-config -o yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: "postgres-svc"
  LOG_LEVEL: "INFO"
  APP_MODE: "production"
```

데이터가 **평문 그대로** 저장됩니다. 비밀이 아닌 값만 넣어야 하는 이유입니다.

## ConfigMap 을 Pod 에 주입하는 두 가지 방법

ConfigMap 을 만들었으면 Pod 안으로 넣어야 합니다. 방법은 두 가지입니다.

### 방법 1 — 환경 변수로 주입

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: my-app
    image: my-app:1.0
    envFrom:
    - configMapRef:
        name: app-config    # ConfigMap 의 모든 key-value 를 환경 변수로
```

Pod 안에서 확인하면:

```bash
kubectl exec my-app -- env | grep -E "DB_HOST|LOG_LEVEL"
# DB_HOST=postgres-svc
# LOG_LEVEL=INFO
```

특정 키만 골라서 넣을 수도 있습니다:

```yaml
env:
- name: DATABASE_HOST       # Pod 안에서 쓸 환경 변수 이름
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: DB_HOST           # ConfigMap 의 키
```

Docker 에서 `docker run -e DB_HOST=postgres` 하던 것과 같은 결과입니다. 다만 K8s 에서는 설정값이 ConfigMap 이라는 별도 리소스에 저장되어 있으니, 이미지를 다시 빌드할 필요 없이 ConfigMap 만 바꾸면 됩니다.

### 방법 2 — 볼륨으로 마운트

설정 **파일** 이 필요한 경우에 씁니다. `nginx.conf`, `application.yml` 같은 파일을 Pod 안에 마운트합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: config-volume
      mountPath: /etc/nginx/conf.d    # 이 경로에 파일이 생긴다
  volumes:
  - name: config-volume
    configMap:
      name: nginx-conf                # ConfigMap 이름
```

ConfigMap 의 각 key 가 **파일 이름**, value 가 **파일 내용**이 됩니다. Pod 안에서 `/etc/nginx/conf.d/nginx.conf` 파일로 접근할 수 있습니다.

### 어떤 방법을 쓸까?

| 방법 | 적합한 경우 |
|---|---|
| **환경 변수** | 간단한 key-value 설정 (`DB_HOST`, `LOG_LEVEL`) |
| **볼륨 마운트** | 설정 파일 통째로 필요할 때 (`nginx.conf`, `application.yml`) |

대부분의 앱은 환경 변수로 충분합니다. 설정 파일이 필요한 경우에만 볼륨 마운트를 씁니다.

## Secret — 비밀을 담는 통

**Secret** 은 ConfigMap 과 구조가 거의 같지만, **비밀 데이터**(패스워드, API 키, TLS 인증서)를 담기 위한 리소스입니다.

### Secret 만들기

```bash
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=my-super-secret-pw \
  --from-literal=API_KEY=sk-abc123xyz
```

또는 YAML 로:

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: bXktc3VwZXItc2VjcmV0LXB3     # Base64 인코딩
  API_KEY: c2stYWJjMTIzeHl6                   # Base64 인코딩
```

YAML 로 만들 때는 값을 **Base64 로 인코딩**해서 넣어야 합니다:

```bash
echo -n "my-super-secret-pw" | base64
# bXktc3VwZXItc2VjcmV0LXB3
```

`stringData` 를 쓰면 평문으로 작성할 수 있습니다 — K8s 가 저장할 때 자동으로 인코딩합니다:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_PASSWORD: "my-super-secret-pw"    # 평문 OK
  API_KEY: "sk-abc123xyz"
```

### Secret 을 Pod 에 주입

ConfigMap 과 방식이 똑같습니다.

```yaml
# 환경 변수로
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: DB_PASSWORD

# 또는 전체를 한 번에
envFrom:
- secretRef:
    name: db-secret
```

볼륨 마운트도 동일합니다 — TLS 인증서 파일을 마운트할 때 자주 씁니다.

## Secret 은 진짜 비밀인가? — 솔직한 이야기

여기서 중요한 질문 하나. **Secret 이 정말 안전할까요?**

솔직히 말하면, 기본 상태에서는 **그렇게 안전하지 않습니다.**

**1. Base64 는 암호화가 아니다**

```bash
echo "bXktc3VwZXItc2VjcmV0LXB3" | base64 -d
# my-super-secret-pw
```

Base64 는 누구나 디코딩할 수 있습니다. 인코딩일 뿐, 암호화가 아닙니다.

**2. etcd 에 평문으로 저장될 수 있다**

기본 설정에서 Secret 은 etcd(1편에서 배운 클러스터 저장소)에 **암호화 없이** 저장됩니다. etcd 에 접근할 수 있는 사람은 Secret 을 볼 수 있습니다.

**3. 그러면 Secret 은 왜 쓰나?**

ConfigMap 과 Secret 을 구분하는 이유는:

- **접근 제어(RBAC)** — Secret 에 대한 읽기 권한을 별도로 관리할 수 있다. ConfigMap 은 누구나 볼 수 있지만, Secret 은 권한이 있는 사람만
- **감사(Audit)** — Secret 접근 로그를 별도로 추적 가능
- **관행** — 비밀을 ConfigMap 에 넣으면 `kubectl get configmap -o yaml` 로 아무나 볼 수 있다

**실무에서 Secret 을 안전하게 쓰려면:**

| 방법 | 설명 |
|---|---|
| **etcd 암호화** | K8s 에서 Encryption at Rest 를 활성화하면 etcd 에 암호화해서 저장 |
| **외부 비밀 관리 도구** | AWS Secrets Manager, HashiCorp Vault, Sealed Secrets 등과 연동 |
| **RBAC 설정** | Secret 읽기 권한을 최소한으로 제한 |
| **Git 에 Secret YAML 을 올리지 않는다** | `.gitignore` 에 추가하거나, Sealed Secrets 로 암호화해서 올린다 |

가장 중요한 건 **Secret YAML 을 Git 에 그대로 올리지 않는 것**입니다. Base64 인코딩된 값은 누구나 디코딩할 수 있으니까요.

## ConfigMap vs Secret — 비교

| | ConfigMap | Secret |
|---|---|---|
| **용도** | 일반 설정 | 비밀 데이터 |
| **저장 형태** | 평문 | Base64 인코딩 (암호화 아님) |
| **RBAC** | 일반 리소스 권한 | 별도 접근 제어 가능 |
| **주입 방법** | 환경 변수 / 볼륨 마운트 | 환경 변수 / 볼륨 마운트 (동일) |
| **크기 제한** | 1 MB | 1 MB |
| **kubectl 출력** | 값이 그대로 보임 | 값이 마스킹됨 (`kubectl get secret` 시) |

사용 기준은 단순합니다: **패스워드, 토큰, 키 → Secret. 나머지 → ConfigMap.**

## ConfigMap 과 Secret 변경 시 주의점

ConfigMap 이나 Secret 을 업데이트하면, 이미 돌고 있는 Pod 에 **자동으로 반영될까요?**

- **환경 변수로 주입한 경우** — 반영 안 됨. Pod 를 재시작해야 합니다
- **볼륨으로 마운트한 경우** — 일정 시간(보통 수십 초~수 분) 뒤에 자동 반영

환경 변수 방식은 Pod 시작 시점에 값을 읽어 오기 때문에, 변경 후 Pod 를 재시작해야 합니다:

```bash
kubectl rollout restart deployment/my-app
```

이 명령은 Deployment 의 Pod 를 롤링 업데이트 방식으로 하나씩 재시작합니다.

## 함정 — 자주 하는 실수

**1. Secret YAML 을 Git 에 커밋한다**

```yaml
data:
  DB_PASSWORD: bXktc3VwZXItc2VjcmV0LXB3   # Base64 일 뿐, 디코딩 가능!
```

이 파일을 Git 에 올리면 비밀번호가 공개됩니다. `stringData` 로 작성한 경우는 더 위험합니다 — 평문이 그대로 보입니다.

**2. ConfigMap/Secret 이름 오타**

```yaml
envFrom:
- configMapRef:
    name: app-confg    # 오타! → Pod 가 시작 실패
```

ConfigMap 이나 Secret 이름이 잘못되면 Pod 가 `CreateContainerConfigError` 상태로 멈춥니다. `kubectl describe pod` 로 확인하면 원인이 바로 보입니다.

**3. 필수 설정을 optional 로 두면**

기본적으로 참조하는 ConfigMap/Secret 이 없으면 Pod 시작이 실패합니다. `optional: true` 를 붙이면 없어도 시작되지만, 앱이 필수 설정 없이 도는 것이므로 의도한 동작인지 확인해야 합니다.

## 가져갈 한 줄

> **이미지는 하나, 설정은 ConfigMap 에, 비밀은 Secret 에 — 다만 Secret 의 "비밀" 은 기본 상태에서 그리 단단하지 않다.**

Docker 에서 `-e` 와 `.env` 로 하던 일이 K8s 에서는 ConfigMap 과 Secret 으로 체계화됩니다. 다만 비밀 관리는 K8s 기본 기능만으로는 부족합니다. 운영에서는 Vault 같은 외부 도구와 조합하는 것이 일반적입니다.

다음 편에서는 **데이터 영속화** 를 다룹니다. 컨테이너가 죽어도 살아남아야 하는 데이터를 어떻게 관리하는가 — Volume, PersistentVolume, PersistentVolumeClaim. Docker 시리즈 4편에서 다뤘던 볼륨의 K8s 버전입니다.

---

## 참고

- [Kubernetes 공식 문서 — ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes 공식 문서 — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Kubernetes 공식 문서 — Encrypting Confidential Data at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)
- [Docker 시리즈 5편 — Dockerfile 심화](/posts/dockerfile-deep-dive/)
