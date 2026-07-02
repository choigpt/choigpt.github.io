---
title: "Nginx"
source: "Notion local cache"
notion_url: "https://app.notion.com/p/364779d1d7da8055ba33ce64b8a6f2a6"
imported_at: "2026-07-02"
tags:
  - notion-import
  - backend-developer
---

# Nginx

## 목차

  - 왜 Nginx를 공부하게 되었나
  - Nginx란 무엇인가
  - Reverse Proxy
  - 500, 502, 504 차이
  - 502가 발생하는 대표 상황
  - 504가 발생하는 대표 상황
  - ADsP 502 경험과 연결하기
  - Nginx 로그에서 봐야 할 것
  - Timeout 설정
  - Upstream 설정
  - Nginx와 애플리케이션 서버의 책임 분리
  - 내가 이 경험에서 배운 것

> ADsP iPad 앱에서 본 `Ubuntu nginx` 502 오류를 계기로 Nginx의 위치와 역할을 정리합니다.
>
> 목표는 “Nginx는 웹서버다”에서 멈추지 않고, **왜 502가 Nginx 화면으로 보였는지**, **Nginx와 애플리케이션 서버 사이에서 어떤 일이 있었을 수 있는지**를 이해하는 것입니다.

## 왜 Nginx를 공부하게 되었나

ADsP 문제풀이 앱을 사용하던 중 다음과 같은 현상을 경험했다.

```
문제풀이 화면 진입이 느려짐
-> 다음 문제 이동도 지연됨
-> 어느 순간 Ubuntu nginx 오류 페이지가 표시됨
-> 잠시 뒤 다시 정상화됨
```

처음에는 단순히 앱이 느리거나 서버가 멈췄다고 생각할 수 있습니다. 하지만 오류 화면에 `nginx`가 보였다는 점이 중요합니다.

사용자가 직접 호출한 것은 앱이지만, 실제 요청 경로 중간에는 Nginx가 있었습니다. Nginx가 upstream 애플리케이션 서버로부터 정상 응답을 받지 못했기 때문에 오류 페이지를 반환했을 가능성이 있습니다.

이 경험을 통해 Nginx를 다음 관점에서 공부할 필요가 생겼다.

- Nginx는 어디에 위치하는가
- Reverse Proxy는 무엇인가
- 502, 504, 500은 어떻게 다른가
- Nginx access log와 error log에서는 무엇을 봐야 하는가
- upstream 서버가 느리거나 죽으면 Nginx는 어떻게 반응하는가
- timeout, connection reset, upstream unavailable은 어떻게 구분하는가

## Nginx란 무엇인가

Nginx는 고성능 웹 서버이면서 reverse proxy, load balancer, static file server, TLS termination, caching layer 역할을 할 수 있는 서버다.

일반적인 백엔드 서비스에서 Nginx는 애플리케이션 서버 앞단에 배치된다.

```
Client
-> Nginx
-> Application Server(Spring Boot, Node.js, Django 등)
-> Database
```

사용자는 Nginx를 직접 의식하지 않는다. 브라우저나 앱은 API 서버에 요청을 보낸다고 생각하지만, 실제로는 Nginx가 요청을 먼저 받고, 내부 애플리케이션 서버로 전달한다.

## Reverse Proxy

Reverse Proxy는 클라이언트 요청을 대신 받아 내부 서버로 전달하는 서버다.

```
Client -> Reverse Proxy -> Upstream Server
```

Nginx를 reverse proxy로 사용하면 다음 장점이 있다.

- 외부에는 Nginx만 노출하고 내부 앱 서버는 숨길 수 있다.
- 여러 애플리케이션 서버로 요청을 분산할 수 있다.
- TLS 인증서 처리를 Nginx에서 담당할 수 있다.
- 정적 파일은 Nginx가 직접 빠르게 응답할 수 있다.
- 요청/응답 timeout, body size, header size 등을 제어할 수 있다.
- access log와 error log를 통해 요청 흐름을 관찰할 수 있다.

예시 설정은 다음과 같다.

```
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://app:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

여기서 `app:8080`이 upstream 애플리케이션 서버다. Nginx는 클라이언트와 직접 통신하고, 앱 서버와도 별도로 통신한다.

## 500, 502, 504 차이

Nginx를 이해할 때 가장 먼저 구분해야 하는 것은 500, 502, 504다.

```
500 Internal Server Error
-> 애플리케이션 서버가 요청을 받았고, 내부 오류를 응답했다.

502 Bad Gateway
-> Nginx가 upstream 서버로부터 유효한 응답을 받지 못했다.

504 Gateway Timeout
-> Nginx가 upstream 서버에 요청을 보냈지만, 정해진 시간 안에 응답을 받지 못했다.
```

운영 관점에서는 질문을 바꿔야 합니다.

ADsP 앱에서 본 `Ubuntu nginx` 오류 화면은 애플리케이션 자체 화면이 아니라 Nginx가 반환한 오류 페이지였을 가능성이 높습니다. 따라서 “앱이 왜 502를 냈는가?”보다 “Nginx가 왜 upstream 앱 서버에서 유효한 응답을 받지 못했는가?”를 먼저 봐야 합니다.

## 502가 발생하는 대표 상황

Nginx의 502는 보통 다음 상황에서 발생한다.

### 1. Upstream 애플리케이션이 내려가 있음

```
Client -> Nginx -> app:8080
                  X app process down
```

Nginx는 `app:8080`으로 연결하려 하지만 연결할 서버가 없다. 이 경우 error log에는 다음과 비슷한 메시지가 남을 수 있다.

```
connect() failed (111: Connection refused) while connecting to upstream
```

### 2. Upstream이 재시작 중임

배포, 장애 복구, Docker restart, Kubernetes restart 등으로 앱이 재시작되는 짧은 순간에도 502가 발생할 수 있다.

```
load pressure
-> app restart
-> Nginx cannot get valid upstream response
-> 502
-> app starts again
-> service recovers
```

사용자 입장에서는 “잠깐 안 되다가 다시 됨”으로 보인다.

### 3. Upstream이 연결을 조기 종료함

Nginx가 앱 서버에 요청을 보냈지만, 앱이 응답을 끝까지 보내기 전에 연결을 닫으면 502가 날 수 있다.

```
upstream prematurely closed connection while reading response header from upstream
```

이는 앱 crash, process kill, timeout, thread interruption 등과 연결될 수 있다.

### 4. 잘못된 upstream 설정

`proxy_pass` 대상이 잘못되었거나, Docker compose network 이름이 틀렸거나, 포트가 다르면 Nginx는 upstream에 연결할 수 없다.

```
proxy_pass http://app:8080;
```

앱 컨테이너 이름이 `adsp-app`인데 Nginx 설정은 `app`을 보고 있다면 DNS resolve 또는 connection 문제가 발생할 수 있다.

## 504가 발생하는 대표 상황

504는 upstream이 죽었다기보다는, 너무 오래 걸려서 Nginx가 기다리다 포기하는 경우에 가깝다.

```
Client
-> Nginx
-> Application Server
-> DB query slow
-> response delayed
-> Nginx proxy_read_timeout exceeded
-> 504
```

error log에는 다음과 비슷한 메시지가 남을 수 있다.

```
upstream timed out while reading response header from upstream
```

DB connection pool이 고갈되거나, 애플리케이션 thread가 꽉 차거나, long query가 발생하면 504가 나타날 수 있다.

ADsP 재현 실험에서도 DB pool을 제한하고 지연을 넣었을 때는 502보다 500/504가 더 직접적으로 나타났다.

## ADsP 502 경험과 연결하기

ADsP 앱 경험을 Nginx 관점에서 다시 쓰면 다음과 같다.

```
시험 전날 사용자 증가
-> 문제 전환마다 서버 API 호출
-> 요청 수 증가
-> DB pool / app thread / CPU / memory 압박
-> 응답 지연
-> 일부 요청은 500 또는 504로 실패 가능
-> 그 상태에서 앱 재시작, crash, connection reset 같은 짧은 upstream interruption 발생
-> Nginx가 502 반환
-> 앱 복구 후 정상화
```

핵심은 502 자체가 “DB가 느리다”의 직접 증거는 아니라는 점이다. DB 지연은 보통 500 또는 504로 드러나기 쉽다. 502는 그보다 한 단계 더 나아가 upstream 앱이 유효한 응답을 주지 못한 상태를 의미한다.

따라서 올바른 사고 흐름은 다음과 같다.

```
느림이 있었다
-> resource pressure 가능성
-> 500/504 가능성

502가 있었다
-> upstream unavailable/reset/restart 가능성

느림 + 502 + 빠른 복구가 같이 있었다
-> 부하 중 짧은 upstream interruption 가능성
```

## Nginx 로그에서 봐야 할 것

502/504를 구분하려면 access log와 error log를 함께 봐야 한다.

### Access Log

access log는 요청 단위로 상태 코드와 응답 시간을 확인할 수 있다.

예시 포맷:

```
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" '
                'rt=$request_time uct=$upstream_connect_time '
                'uht=$upstream_header_time urt=$upstream_response_time';
```

여기서 중요한 값은 다음과 같다.

```
$status                  -> 500, 502, 504 등 최종 응답 코드
$request_time            -> 클라이언트 요청 전체 처리 시간
$upstream_connect_time   -> upstream 연결에 걸린 시간
$upstream_header_time    -> upstream 응답 헤더를 받기까지 걸린 시간
$upstream_response_time  -> upstream 응답 전체 시간
```

### Error Log

error log는 왜 실패했는지를 더 직접적으로 보여준다.

대표 메시지는 다음과 같다.

```
connect() failed (111: Connection refused) while connecting to upstream
```

앱 서버가 떠 있지 않거나 포트가 닫혀 있을 가능성이 있다.

```
upstream timed out while reading response header from upstream
```

upstream이 너무 늦게 응답해서 504로 이어질 가능성이 크다.

```
upstream prematurely closed connection while reading response header from upstream
```

upstream이 응답을 완성하기 전에 연결을 닫았다. 앱 crash, restart, connection reset 등을 의심할 수 있다.

## Timeout 설정

Nginx reverse proxy에서 자주 보는 timeout 설정은 다음과 같다.

```
location / {
    proxy_pass http://app:8080;

    proxy_connect_timeout 3s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}
```

각 설정의 의미는 다음과 같습니다.

- `proxy_connect_timeout`: upstream 서버와 연결을 맺을 때까지 기다리는 시간
- `proxy_send_timeout`: Nginx가 upstream으로 요청을 보내는 동안 기다리는 시간
- `proxy_read_timeout`: upstream 응답을 읽을 때까지 기다리는 시간

주의할 점은 timeout을 무작정 늘리는 것이 해결책이 아니라는 점입니다. timeout을 늘리면 504는 줄어 보일 수 있지만, 느린 요청이 더 오래 붙잡혀 thread, connection, memory를 더 오래 점유할 수 있습니다.

## Upstream 설정

여러 앱 서버가 있다면 `upstream` 블록으로 묶을 수 있다.

```
upstream adsp_app {
    server app1:8080;
    server app2:8080;
}

server {
    listen 80;

    location / {
        proxy_pass http://adsp_app;
    }
}
```

이 구조에서는 한 서버가 죽어도 다른 서버로 트래픽을 보낼 수 있다. 하지만 health check, retry 정책, 배포 방식이 적절하지 않으면 여전히 502가 사용자에게 보일 수 있다.

## Nginx와 애플리케이션 서버의 책임 분리

Nginx는 요청을 빠르게 받고 전달하는 데 강하다. 하지만 비즈니스 로직을 직접 처리하지는 않는다.

```
Nginx 책임:
- client connection 처리
- reverse proxy
- static file serving
- TLS termination
- timeout / body size 제한
- access log / error log 기록

Application 책임:
- API 비즈니스 로직
- DB 접근
- 인증/인가
- 문제 데이터 조회
- 답안 저장
- 예외 처리
```

장애 분석 시에는 “Nginx가 502를 냈다”에서 멈추면 안 된다. Nginx는 결과를 보여준 계층이고, 원인은 upstream 앱 서버, DB, 배포, 네트워크, 리소스 제한 중 하나일 수 있다.

## 내가 이 경험에서 배운 것

이번 502 경험을 통해 배운 점은 다음과 같다.

1. Nginx 오류 화면이 보였다는 것은 요청 경로에 reverse proxy가 있었다는 뜻이다.
1. 502는 애플리케이션 내부 예외라기보다 upstream 통신 실패에 가깝다.
1. 느린 응답은 resource pressure를 의심하게 만들고, 504와 연결될 수 있다.
1. 502는 app restart, crash, connection reset 같은 upstream interruption과 더 가깝다.
1. 사용자 입장에서는 하나의 장애처럼 보여도, 운영자는 access log, error log, app log, DB metric을 함께 봐야 한다.
1. 문제별 서버 호출처럼 요청 수를 증폭시키는 구조는 Nginx 앞단에서도 장애 표면을 키운다.
1. Nginx를 이해하면 단순히 “서버가 터졌다”가 아니라, 요청이 어느 계층에서 실패했는지 분해해서 볼 수 있다.

## 정리

Nginx는 단순 웹서버가 아니라, 클라이언트와 애플리케이션 서버 사이에서 요청을 받고 전달하며, 실패 상황을 상태 코드로 드러내는 중요한 관찰 지점이다.

ADsP 앱에서 본 502는 Nginx 자체가 원인이라기보다, Nginx가 upstream 앱 서버와 정상적으로 통신하지 못했다는 신호로 보는 것이 타당하다.

따라서 앞으로 502를 만나면 다음 순서로 봐야 한다.

```
1. access log에서 status, request_time, upstream_response_time 확인
2. error log에서 connection refused / timed out / prematurely closed 확인
3. app log에서 crash, restart, exception 확인
4. DB metric에서 connection pool, query latency 확인
5. 배포/재시작 이력 확인
```

이 순서로 보면 502를 단순 오류 화면이 아니라, reverse proxy와 upstream 서버 사이의 실패 신호로 해석할 수 있다.
