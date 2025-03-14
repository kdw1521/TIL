# Self-hosted Runner (VS. Github-hosted Runner)

### Github-hosted Runner

- 관리 및 환경: Github 에서 인프라를 관리해준다.
- 비용: 무료플랜에서는 한도가 있다.
- 장점
  - 관리 부담 감소: 인프라 등 Github이 처리
  - 빠른 설정
  - 자동확장
- 단점
  - 제한된 커스터마이징
  - 동시 실행 제한: 무료 플랜에서 제한 있음.

### Self-hosted Runner

- 관리 및 환경: 개발자가 관리
- 비용: 클라우드 서버 등 인프라 비용 발생, Github Runner 사용한도는 없음.
- 장점
  - 높은 커스터마이징
  - 비용 효율
- 단점
  - 인프라 등 관리를 개발자가 해야함.

# 준비

- EC2
- Github Self-hosted Runner 설정 (+ .github/workflows/deploy.yml)
- 배포 shell script
- docker-compose.yml
- Dockerfile
- Slack webhook (optional)

# 구현

### 1. EC2 에 Github Self-hosted Runner를 설치 및 등록.

- Github Repository 에서 Settings > Actions > Runners 에 진입
- `New self-hosted runner` 를 클릭
- 해당 하는 OS, Architecture 선택
- 각 단계별 명령어 입력 - Download > Configure
  - working directory 는 \_work 로 지정 (아마 default 값인듯 - 변경도 가능, actions-runner 디렉토리안에 관련 모든게 다운로드 됨.)
  - Configure 에서 run.sh 를 실행할때 백그라운드 에서 실행을 위해 `nohup ./run.sh > runner.log 2>&1 &` 이렇게 실행 - runner.log 이 파일에서 로그를 확인

### 2. .github/workflows/deploy.yml

```yaml
name: [PROD] My-Project Deploy to EC2

# 트리거 조건
on:
  workflow_dispatch: # 수동 실행 트리거
  push:
    branches:
      - main # main 브랜치에서만 작동

jobs:
  deploy:
    runs-on: self-hosted
    defaults:
      run:
        working-directory: /home/ubuntu/actions-runner/_work/my-project

    steps:
      # 1. GitHub 코드 체크아웃
      - name: Checkout Code
        uses: actions/checkout@v3

      # 2. Runner 디렉토리 권한 변경
      - name: Change runner permission
        run: |
          sudo chown -R ubuntu:ubuntu /home/ubuntu/actions-runner/_work/my-project

      # 3. Set up environment
      - name: Set up environment
        run: |
          sudo chmod +x blue-green-deploy.sh
          sudo chmod +x nginx-reload.sh

      # 4. Run Blue/Green Deployment script
      - name: Blue/Green Deployment
        run: |
          ./blue-green-deploy.sh

      # 5. Send slack msg
      - name: Notify Slack on Success
        if: success()
        run: |
          curl -X POST -H 'Content-type: application/json' --data '{
            "text": "[My-Project] 🚀 백엔드 배포 완료!",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "🚀 *My-Project 백엔드 배포 성공*\n"
                }
              },
              {
                "type": "section",
                "fields": [
                  {
                    "type": "mrkdwn",
                    "text": "*작업자:*\n${{ github.actor }}"
                  }
                ]
              }
            ]
          }' my-slack-url

      - name: Notify Slack on Failure
        if: failure()
        run: |
          Lcurl -X POST -H 'Content-type: application/json' --data '{
            "text": "[My-Project] 🔴 백엔드 배포 실패!",
            "blocks": [
              {
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "🔴 *My-Project 백엔드 배포 실패*\n"
                }
              },
              {
                "type": "section",
                "fields": [
                  {
                    "type": "mrkdwn",
                    "text": "*작업자:*\n${{ github.actor }}"
                  }
                ]
              }
            ]
          }' my-slack-url
```

성공, 실패 등 더 자세한 로그가 필요하면 추가를 해주면 된다.

### 3. blue-green-deploy.sh, nginx-reload.sh 작성

```bash
# blue-green-deploy.sh

#!/bin/bash

# 1. 현재 활성화된 환경 확인 (파일에서 읽기) - _env/active-env.txt 는 개발자가 추가
current_env=$(cat /home/ubuntu/actions-runner/_env/active-env.txt)

# 2. 다음 배포 환경 및 포트 결정
if [ "$current_env" == "blue" ]; then
    next_env="green"
    next_port="1324"
    current_port="1323"
else
    next_env="blue"
    next_port="1323"
    current_port="1324"
fi

echo "[현재 환경]: $current_env (Port: $current_port)"
echo "[실행될 환경]: $next_env (Port: $next_port)"

# 3. 반대 환경(Green 또는 Blue)에 새 버전 배포
sudo docker-compose -f docker-compose-prod.yml up -d --build be-app-$next_env

# 4. 새 환경 준비 상태 확인
echo "[대기상태]: $next_env 환경 health 체크 중..."
success=false  # 초기 상태: 실패로 설정
for i in {1..10}; do
    if curl -f http://localhost:$next_port/health; then # health api 는 백엔드 프로젝트에 추가
        echo "$next_env environment is ready."
        success=true # 성공 플래그 설정
        break
    fi
    echo "Waiting..."
    sleep 5
done

# 4-1. Health check 실패 시 배포 중단
if [ "$success" == "false" ]; then
    echo "🚨 Health Check Failed! 배포 중단 및 기존 서비스 유지"

    # 🚨 실패 시 새 컨테이너 중지 및 삭제
    sudo docker-compose -f docker-compose-prod.yml stop be-app-$next_env
    sudo docker-compose -f docker-compose-prod.yml rm -f be-app-$next_env

    # 🚨 Slack 알림
    curl -X POST -H 'Content-type: application/json' --data '{
        "text": "⚠️ [배포 실패] Health Check 실패! 기존 서비스 유지"
    }' my-slack-url

    exit 1  # 배포 실패 처리
fi

# 5. Nginx 설정 업데이트 (트래픽 전환)
echo "[Nginx 스위칭]: $next_env environment..."
sed -i "s/my-project-$current_env:1323/my-project-$next_env:1323/" ./nginx.conf # upstream 전환

# 5-1. Nginx 설정 테스트 실패시 nginx.conf 기존으로 되돌리기위해 환경 export
export CURRENT_ENV=$current_env
export NEXT_ENV=$next_env

# 5-2. Nginx 설정 반영 스크립트 호출
./nginx-reload.sh

# 6. 기존 환경(Blue 또는 Green) 컨테이너 종료 - 기존요청이 있을수 있어 3초 대기
sleep 3
sudo docker-compose -f docker-compose-prod.yml stop be-app-$current_env
sudo docker-compose -f docker-compose-prod.yml rm -f be-app-$current_env

# 7. 새로운 활성화 상태 기록
echo "$next_env" > /home/ubuntu/actions-runner/_env/active-env.txt

echo "[Blue/Green 배포완료] [현재 환경]: $next_env (Port: $next_port)"
```

```bash
# nginx-reload.sh

#!/bin/bash

echo "[Nginx 설정 업데이트]"

# 1. Nginx 설정 테스트
echo "Nginx 설정 파일 테스트 중..."
if ! docker exec nginx nginx -t; then
    echo "Nginx 설정 파일 테스트 실패. 변경 내용을 확인하세요. nginx.conf를 기존으로 돌립니다."
    sed -i "s/my-project-${NEXT_ENV}:1323/my-project-${CURRENT_ENV}:1323/" ./nginx.conf
    exit 1
fi

# 2. Nginx 재시작
echo "Nginx 재시작 중..."
sudo docker-compose -f docker-compose-prod.yml restart nginx

echo "[Nginx 설정 업데이트 완료]"
```

\_env/active-env.txt 에는 현재 실행중인 환경이 적혀있고 (최초 blue 또는 green 값을 개발자가 추가해준다.)  
이걸 기준으로 다음 환경을 할당해 배포를 진행한다.

### 4. docker-compose-prod.yml, Dockerfile-prod 작성

```Dockerfile
# docker-compose-prod.yml

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

  be-app-green:
    build:
      context: .
      dockerfile: Dockerfile-prod
    image: my-project:prod
    container_name: my-project-green
    ports:
      - "1324:1323" # Green 컨테이너의 내부 포트
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
      - be-app-green
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

```Dockerfile
# Dockerfile-prod

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

docker-compose 에는 Dockerfile-prod 를 기반으로 blue, green 2가지 컨테이너를 둔다.

# 배포 진행

### 1. github repository 의 main 브랜치에 pr을 통해 merge를 진행

EC2 에서 github runner job 을 polling으로 체크해 github/workflows/deploy.yml 에 설정한대로 배포를 진행한다.

# 이슈

### 간헐적으로 job을 가져오지 못함.

runner 관련 프로세스를 중지 후 재시작하여 해결함.

```cmd
$ ps aux | grep -i "run"

...
ubuntu     80892  0.0  0.0   7740  3456 ?        S    Mar07   0:00 /bin/bash ./run.sh
ubuntu     80896  0.0  0.0   7740  3456 ?        S    Mar07   0:00 /bin/bash /home/ubuntu/actions-runner/run-helper.sh
ubuntu     80900  0.0  3.5 274025988 138544 ?    Sl   Mar07   2:47 /home/ubuntu/actions-runner/bin/Runner.Listener run
...
```

위의 3가지 프로세스를 중지 하고 다시 run.sh 을 실행해 정상동작을 확인함.

하지만, 완전한 해결책은 아님. 하단 이슈를 트래킹 해야함.  
[Self-hosted runner Issue - Waiting for a runner to pick up this job...](https://github.com/actions/runner/issues/3609)

# 같이 보면 좋을 링크

- [다단계빌드](https://github.com/kdw1521/TIL/blob/main/Docker/%EB%8B%A4%EB%8B%A8%EA%B3%84%EB%B9%8C%EB%93%9C.md)
- [Docker-compose 를 이용한 nginx,backend 처리](https://github.com/kdw1521/TIL/blob/main/Docker/Docker-compose_%EB%A5%BC_%EC%9D%B4%EC%9A%A9_nginx_upstream.md)
