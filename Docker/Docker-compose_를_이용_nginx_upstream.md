# docker-compose

nginx 와 backend 이미지를 이용해 nginx에서 bankend를 upstream 처리를 하도록 한다.

# 준비

- Docker & Docker-compose
- docker-compose.yml
- NestJS (backend) \*각 api prefix는 /api 로 설정
- nginx.conf

# 구현

### docker-compose-prod.yml

```Dockerfile
services:
  be-app-blue:
    build:
      context: .
      dockerfile: Dockerfile-prod
    image: my-project:prod
    container_name: my-project-blue
    ports:
      - "1323:1323" # Blue 컨테이너의 내부 포트
    env_file:
      - .env.production
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "500m"
        max-file: "2"
    networks:
      - app-network

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80" # 외부 포트
    volumes:
      - "./nginx.conf:/etc/nginx/nginx.conf"
    depends_on:
      - be-app-blue
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "500m"
        max-file: "2"
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

### Dockerfile-prod

```Dockerfile
# 베이스 이미지 설정
FROM node:22-alpine AS base

# 작업 디렉토리 설정
WORKDIR /app

# 의존성 파일 복사 및 설치
COPY package.json yarn.lock .env.production ./
RUN yarn install --frozen-lockfile

# 소스 복사
COPY . .

# 빌드 단계
ENV NODE_ENV=production
RUN yarn prisma:prod
RUN yarn build

# 프로덕션 환경용 설정
FROM node:22-alpine AS production

# 작업 디렉토리 설정
WORKDIR /app

# 프로덕션 빌드 파일 복사
COPY --from=base /app/dist ./dist
COPY --from=base /app/node_modules ./node_modules
COPY package.json ./

# 실행 명령 및 포트 노출
EXPOSE 1323

CMD ["yarn", "start:prod"]
```

### nginx.conf

```nginx
events {
  worker_connections 1024; # 연결을 처리할 워커의 최대 연결 수
}

http {
  keepalive_timeout 300s;  # 클라이언트와의 연결 유지 시간
  proxy_read_timeout 30s; # 백엔드에서 데이터를 읽는 타임아웃
  upstream backend {
    server my-project-blue:1323;
  }

  server {
    listen 80;

    location /api {
      proxy_pass http://backend;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection 'keep-alive';
      proxy_set_header Host $host;
      proxy_cache_bypass $http_upgrade;
    }

    # 기본 루트 설정
    location / {
      default_type text/plain;
      return 200 'OK';
    }
  }
}
```

nginx.conf는 volume 연결을 해두었고, 각 컨테이너별 로그 파일 사이즈와 개수를 제한해 두었다.

# 같이 보면 좋을 링크

- [다단계빌드](https://github.com/kdw1521/TIL/blob/main/Docker/%EB%8B%A4%EB%8B%A8%EA%B3%84%EB%B9%8C%EB%93%9C.md)
- [Self-hosted runner](https://github.com/kdw1521/TIL/blob/main/Github/Actions/Self-hosted-Runner_%EB%A1%9C_blue-green.md)
