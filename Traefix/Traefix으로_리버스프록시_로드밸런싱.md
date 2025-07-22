# Traefik 으로 리버스 프록시 와 로드밸런싱 처리를 하자.

채팅 백엔드를 구성하며, 리버스프록시와 LB 처리를 위해 Traefik 을 이용해 구성해본다.

Traefik 이란?

```text
HTTP 및 TCP 기반 애플리케이션을 위한 오픈 소스 리버스 프록시이자 로드 밸런서.
클라이언트 요청을 받아 내부 서비스로 전달하고, 여러 인스턴스에 부하를 분산시키는 데 특화되어 있음.
```

전통적인 웹서버와의 차이?

```text
전통적인 웹 서버는 정적 파일 서빙(static content)과 다양한 모듈(예: PHP, FastCGI) 지원을 통해 동적 콘텐츠를 처리하는 데 최적화되어 있음.
반면 Traefik은 서비스 디스커버리(service discovery)와 동적 라우팅(dynamic routing)에 중점을 두어, 컨테이너 오케스트레이션 환경에서 자동으로 설정을 감지·적용함.
```

# 준비

- EC2
- Route53 (도메인이 준비되어 있음)
- BE

# 구현

## 1. docker 작성

```Docker
# builder
FROM node:22-alpine AS builder
WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install --frozen-lockfile

COPY . .

ENV NODE_ENV=dev
RUN yarn build

# runner
FROM node:22-alpine AS runner
ENV NODE_ENV=dev
WORKDIR /app

COPY --from=builder /app/.env.dev ./
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

EXPOSE 8002

CMD ["node", "dist/main.js"]
```

## 2. docker-compose 작성

개발용이기에 redis 도 Docker 로 구동.

```Docker
services:
  traefik:
    image: traefik:3.4
    container_name: traefik
    restart: unless-stopped
    command:
      - --api.insecure=true # 대시보드 활성화 (개발용)
      - --providers.docker=true # Docker provider 사용
      - --entrypoints.web.address=:80 # HTTP 엔트리포인트
      - --entrypoints.web.http.redirections.entrypoint.to=websecure # HTTP -> HTTPS
      - --entrypoints.web.http.redirections.entrypoint.scheme=https # 리다이렉트 시 HTTPS 프로토콜로 지정
      - --entrypoints.websecure.address=:443 # HTTPS 엔트리포인트
      - --entrypoints.websecure.http.tls=true # HTTPS 사용 명시
      - --certificatesresolvers.le.acme.email=kdw1521@naver.com # Let's Encrypt 등록 이메일
      - --certificatesresolvers.le.acme.storage=/letsencrypt/acme.json # ACME 인증서와 키를 저장할 경로를 지정
      - --certificatesresolvers.le.acme.httpchallenge.entrypoint=web # HTTP-01 챌린지를 web 엔트리포인트(포트 80)에서 처리하도록 설정
    ports:
      - '80:80' # (HTTP)
      - '443:443' # (HTTPS)
      - '8080:8080' # (Traefik 대시보드)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro # Docker 소켓을 읽기 전용으로 마운트해 컨테이너 생성·삭제 이벤트를 감지
      - traefik-certs:/letsencrypt # 인증서와 키를 영속화하기 위해 named volume을 사용
    logging:
      driver: json-file
      options:
        max-size: '10m' # 로그 파일 하나당 최대 10MB
        max-file: '2' # 최대 2개 파일

  redis:
    image: redis:7-alpine
    container_name: dev-chat-redis
    restart: always
    ports:
      - '6379:6379'
    volumes:
      - redis-data:/data
    logging:
      driver: json-file
      options:
        max-size: '10m' # 로그 파일 하나당 최대 10MB
        max-file: '2' # 최대 2개 파일

  backend:
    build:
      context: .
      dockerfile: Dockerfile-dev
    restart: always
    env_file:
      - .env.dev
    expose:
      - '8002' # Traefik이 자동 감지 가능하도록
    labels:
      - 'traefik.enable=true' # 해당 컨테이너를 Traefik 라우팅 대상에 포함
      - 'traefik.http.routers.devchat.rule=Host(`dev-chat.wando.com`)' # 도메인 기반 라우팅 룰을 설정
      - 'traefik.http.routers.devchat.entrypoints=websecure' # HTTPS 엔트리포인트를 사용하도록 지정
      - 'traefik.http.routers.devchat.tls.certresolver=le' # Let's Encrypt로 인증서를 발급받도록 설정
      - 'traefik.http.services.devchat.loadbalancer.server.port=8002' # Traefik이 컨테이너 내부의 8002 포트로 트래픽을 전달하도록 지정
    depends_on:
      - redis
      - traefik
    logging:
      driver: json-file
      options:
        max-size: '10m' # 로그 파일 하나당 최대 10MB
        max-file: '2' # 최대 2개 파일

volumes:
  traefik-certs: # Traefik의 ACME 인증서 저장용 볼륨
  redis-data: # Redis 데이터 영속화를 위한 볼륨
```

## 3. Traefik 로드 밸런싱

위와 같은 구조로 진행하고, 백엔드 서비스를 2개를 띄워본다.

```sh
docker compose -f docker-compose-dev.yaml up -d --no-deps --build --scale backend=2 backend
```

이렇게 진행하면,  
Traefik은 /var/run/docker.sock 을 읽어 Docker 이벤트를 모니터링하며, labels에 지정된 라우팅 규칙(Host, entrypoint, port 등)을 가진 컨테이너를 자동으로 감지한다.  
그리고 Traefik 아키텍처상 Service 레이어가 다수 인스턴스 사이에 트래픽을 분산하게 된다. (기본값인 Weighted Round Robin(WRR)을 사용)

```text
Traefik 아키텍처

Provider: 서비스 인스턴스(컨테이너)의 IP와 포트를 수집
Entrypoint: 클라이언트 요청 수신(포트 80/443)
Router: 요청의 호스트·경로 매칭
Service: 실제 백엔드 서버 그룹, 로드 밸런서 기능 포함
Middleware: 요청 전·후 처리
```
