# 그동안 스터디한 내용을 총집합해 사용해보자.

NestJS(pod) 에서 prisma 를 이용하고, Mysql(pod) 에 연결해본다.  
Mysql(pod) 는 k8s 내부에서만 접근가능하도록 하고, port forward를 이용해서 로컬에서 접속해본다.  
Pod만을 이용하기 때문에 이번에는 로컬에서 작업하고 이런건 배제했다.

# 준비

- Mysql (pod)
- NestJS + Prisma (pod)

# 구현

### 1. NestJS + Prisma Docker Image를 세팅해준다.

```bash
> npm install prisma --save-dev
> npx prisma init
```

1.1 schema.prisma 를 수정해준다. (defualt 로 postgresql 이기 때문에)

```javascript
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mysql" // 이부분 변경
  url      = env("DATABASE_URL")
}
```

1.2 .env 를 수정해준다.

```text
DATABASE_URL="mysql://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}"
```

1.3 Dockerfile 을 작성해준다.  
Prisma를 사용하기 때문에 이 프로젝트는 이미 구축되어있는 DB 를 pull 받아서 generate하는 전제로 진행한다. 그렇기에 entrypoint shell 을 하나 만들어 진행한다.

1.3.1 entrypoint.sh 작성

```bash
#!/bin/sh
set -e

# Prisma 스키마 동기화 및 클라이언트 생성 (환경 변수 주입 후 실행됨)
npx prisma db pull
npx prisma generate

# 애플리케이션 시작
exec node dist/main.js
```

1.3.2 Dockerfile 작성

```dockerfile
FROM node

WORKDIR /app

COPY . .

RUN npm install
RUN npm run build

# entrypoint 스크립트를 컨테이너로 복사 및 실행 권한 부여
COPY entrypoint.sh /app/entrypoint.sh
RUN chmod +x /app/entrypoint.sh

EXPOSE 3000

ENTRYPOINT [ "/app/entrypoint.sh" ]
```

1.3.3 Docker Image 생성 (nest-server 이름으로 지정한다.)

```bash
> docker build -t nest-server:1.0 .
```

### 2. NestJS 의 Deployment, Service, ConfigMap, Secret 을 작성해 연결한다.

2-1. nest-deployment.yaml 작성

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nest-deployment

spec:
  replicas: 3
  selector:
    matchLabels:
      app: be-app

  template:
    metadata:
      labels: be-app
    spec:
      containers:
        - name: nest-container
          image: nest-server:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000

          env:
            - name: DB_HOST # 소스와 매핑 값
              valueFrom:
                configMapKeyRef:
                  name: nest-config
                  key: db-host
            - name: DB_PORT
              valueFrom:
                configMapKeyRef:
                  name: nest-config
                  key: db-port
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: nest-config
                  key: db-name
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: nest-secret
                  key: db-username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: nest-secret
                  key: db-password
```

2-2. nest-service.yaml 작성

```yaml
apiVersion: v1
kind: Service

metadata:
  name: nest-service

spec:
  type: NodePort
  selector:
    app: be-app
  ports:
    - protocol: TCP
      nodePort: 30000
      port: 3000
      targetPort: 3000
```

2-3. nest-config.yaml 작성

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: nest-config

data:
  db-host: mysql-service # mysql pod에 정의된 이름
  db-port: "3306"
  db-name: kub-practice
```

2-4. nest-secret.yaml 작성

```yaml
apiVersion: v1
kind: Secret

metadata:
  name: nest-secret

stringData:
  db-username: root
  db-password: 123qwe
```

### 3. 기존 Mysql 의 Service 부분을 수정해준다.

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mysql-service

spec:
  type: ClusterIP # k8s 내부만 통신 가능
  selector:
    app: mysql-db
  ports:
    - protocol: TCP
      port: 3306
      targetPort: 3306
```

### 4. 모두 실행해 확인해본다.

```bash
> kubectl apply -y nest-secret.yaml
> kubectl apply -y nest-config.yaml
> kubectl apply -y nest-service.yaml
> kubectl apply -y nest-deployment.yaml
> kubectl apply -y mysql-service.yaml
> kubectl rollout restart deployment nest-deployment
```

1. DB 연결이 안된다면 nest pod 가 실행되지 않는다.
2. 정상적으로 연결되었다면, db 커넥션 GUI 에서 접근을 할수 없다.
3. 로컬에서 db에 접근하려면 port-forward 를 이용한다.

### 5. 로컬에서 DB 를 접속해보자.

```bash
> kubectl port-forward mysql-deployment-6886bc8445-c9l2g 30002:3306
```

mysql-deployment-6886bc8445-c9l2g 이 부분은 파드이름이다.
