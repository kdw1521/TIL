# EC2와 K3S를 이용해 개발서버를 운영해보자.

현재 개발서버 한대에 BE,FE,DB 등 엄청 많은 서비스들이 운영중이라 배포시 프로세스가 죽는 경우가 있어 백엔드 쪽을 우선적으로 옮기기로 결정했다.  
이에 K3S 를 이용해 간단하게 구성해본다.

# 준비

- EC2 (t4g.medium / small은 메모리가 부족했다.)
- ECR
- Route53 (도메인이 준비되어 있음)
- BE, Manifest Repository (GitHub을 이용했다.)

# 구현

## 1. AWS IAM 생성과 로컬 머신을 구성한다음

- AWS Key 들을 생성한다.  
  `Root Key(로컬사용)`, `IAM Key(EC2 사용)` 를 생성한다.

```text
1. Root Key
보안 자격 증명(우측상단 클릭) > 액세스 키 > 액세스 키 만들기

2. IAM
액세스 관리 > 사용자 > 사용자 생성 > 권한에 AmazonEC2ContainerRegistryFullAccess 를 추가해준다.

각 key들은 외부에 노출되지 않게한다.
```

- 로컬 머신에 aws-cli를 설치하고 key를 등록해주자.  
  Root key 로 진행했다. (`절대 노출되어서는 안된다.`)

```bash
> brew install awscli
> aws configure
```

## 2. 백엔드를 구성한다. (Dockerfile, ECR Push Shell)

- Dockerfile-dev (개발용 Dockerfile)

```Dockerfile
# 베이스 이미지 설정
FROM node:22-alpine AS base

# 작업 디렉토리 설정
WORKDIR /app

# 의존성 파일 복사 및 설치
COPY package.json yarn.lock .env.development ./
RUN yarn install --frozen-lockfile

# 소스 복사
COPY . .

# 개발 환경 설정
ENV NODE_ENV=development

# 포트 노출
EXPOSE 1323

# Prisma와 개발 실행 명령
CMD ["sh", "-c", "yarn prisma:dev && yarn start:dev"]
```

- ECR 에 Push 하는 shell script를 작성한다.  
  버전이 있을 경우 체크하는데 개발서버에서 버전이 바뀌면 변경해줘야해서 귀찮아서 나는 latest로만 진행했다.

```bash
#!/bin/bash

# 첫 번째 인자로 버전 값을 받아오고, 전달되지 않으면 기본값 "latest" 사용
VERSION=${1:-"latest"}

# 만약 버전 값이 "latest"가 아니라면, "X.Y" 형식이어야함.
if [ "$VERSION" != "latest" ]; then
  # X.Y 형식인지 확인
  if ! [[ "$VERSION" =~ ^[0-9]+\.[0-9]+$ ]]; then
    echo "Error: Invalid version format. Please use the format X.Y (e.g., 1.1, 2.0, 2.1)."
    exit 1
  fi
fi

echo "Version: $VERSION"

echo "Login ECR"
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin {ECR주소를 넣는다.}

echo "Image build"
docker build -f Dockerfile-dev -t {ECR 폴더 경로를 넣는다.}:${VERSION} .

echo "Apply Tag"
docker tag {ECR 폴더 경로를 넣는다.}:${VERSION} {ECR주소를 넣는다.}/{ECR 폴더 경로를 넣는다.}:${VERSION}

echo "Push To ECR"
docker push {ECR주소를 넣는다.}/{ECR 폴더 경로를 넣는다.}:${VERSION}
```

- deploy.sh 로 위의 ECR push shell 과 remote deploy.sh 을 실행하게 하자.

```bash
#!/bin/bash

sh "$(dirname "$0")/dev-push-ecr.sh"

echo "[remote shell command]"
ssh -i {EC2 pem key를 적어준다.} {ec2 user를 적어준다}@{ec2 ip 를 적어준다.} 'sh {deploy.sh 이 있는 경로를 적어준다.}'
# ex. ssh -i keypair.pem ubuntu@1.1.1.1 'sh /home/ubuntu/deploy.sh'
```

## 3. EC2 를 세팅하자.

- 필요 서비스들을 설치하자.

1. Docker 설치

```bash
$ sudo apt-get update && \
	sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common && \
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - && \
	sudo apt-key fingerprint 0EBFCD88 && \
	sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" && \
	sudo apt-get update && \
	sudo apt-get install -y docker-ce && \
	sudo usermod -aG docker ubuntu && \
	newgrp docker && \
	sudo curl -L "https://github.com/docker/compose/releases/download/2.27.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && \
	sudo chmod +x /usr/local/bin/docker-compose && \
	sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose

$ docker -v # Docker 버전 확인
$ docker compose version # Docker Compose 버전 확인
```

2. k3s 설치

```bash
$ curl -sfL https://get.k3s.io | sh - # k3s 설치
$ sudo chmod 644 /etc/rancher/k3s/k3s.yaml # 권한 부여
$ sudo kubectl version # k3s 잘 설치됐는 지 확인
```

3. aws cli 설치

```bash
$ cd ~
$ sudo apt install unzip
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install
$ aws --version # 잘 출력된다면 정상 설치된 상태
```

4. aws cli key 등록

```bash
$ aws configure

AWS Access Key ID [None]: <위에서 발급한 IAM Key id>
AWS Secret Access Key [None]: <위에서 발급한 IAM Secret Access Key>
Default region name [None]: ap-northeast-2
Default output format [None]:
```

5. k3s 가 ECR로부터 이미지를 받아올 때 인증을 할 수 있도록 Secret 추가

```bash
$ kubectl create secret generic regcred --from-file=.dockerconfigjson=/home/ubuntu/.docker/config.json --type=kubernetes.io/dockerconfigjson

$ kubectl get secret # 생성확인
```

## 4. Route53 을 이용해 EC2로 연결하자.

1. 호스팅 영역에 쓸 도메인에 들어간다.
2. 레코드를 생성한다.
3. A 레코드를 생성해 EC2에 연결한다.

## 5. Manifest 를 생성한다.

k3s 로 인증서를 발급받아 Ingress 로 백엔드 Service를 연결한다. 해당 Manifest 파일은 Git Repository에서 관리한다.

1. 폴더구조

```bash
.
├── cluster-issuer.yaml
├── nest-backend
│   ├── deployment.yaml
│   └── service.yaml
└── multi-service-ingress.yaml
```

2. cluster-issuer.yaml  
   cert-manager 가 Let's Encrypt 의 ACME 프로토콜을 사용해 인증서를 발급받을 때 사용할 정책을 정의한다.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer

metadata:
  name: letsencrypt-prod

spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory # 프로덕션 ACME 서버를 사용
    email: test@gmail.com # 본인의 이메일 주소로 변경
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01: # HTTP-01 챌린지를 사용해 도메인 소유권을 검증, 여기서는 Ingress 리소스를 사용하여 Traefik Ingress Controller를 통해 챌린지 응답을 제공하도록 설정
          ingress:
            class: traefik # k3s 기본 Ingress Controller가 Traefik인 경우. Nginx를 사용 중이면 "nginx"로 변경.
```

3. multi-service-ingress.yaml  
   외부의 HTTPS 요청(예: dev.test.com)에 대해 인증서 기반 TLS 종료를 수행하고, 요청을 클러스터 내부의 백엔드 서비스(my-service)로 라우팅

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: multi-service-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"

spec:
  tls:
    - hosts:
        - dev.test.com # 본인의 도메인을 입력
      secretName: dev-api-tls
  rules:
    - host: dev.test.com # 본인의 도메인을 입력
      http:
        paths:
          - path: /api # prefix 로 지정한 path를 적어준다.
            pathType: Prefix
            backend:
              service:
                name: my-service # Service에 정의한 name을 입력한다.
                port:
                  number: 1323 # Service의 포트번호를 입력한다.
```

4. 백엔드 서비스의 deployment.yaml  
   ECR 이미지를 기반으로 한다.

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: my-be-deployment

spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-be

  template:
    metadata:
      labels:
        app: my-be
    spec:
      imagePullSecrets:
        - name: regcred
      containers:
        - name: my-be-container
          image: ${ECR 주소를 입력한다.}
          imagePullPolicy: Always
          ports:
            - containerPort: 1323
```

5. 백엔드 서비스의 service.yaml
   ingress로 라우팅을 진행하므로 내부에서만 돌도록 설정한다.

```yaml
apiVersion: v1
kind: Service

metadata:
  name: my-service

spec:
  type: ClusterIP
  selector:
    app: my-be
  ports:
    - protocol: TCP
      port: 1323
      targetPort: 1323
```

## 6. EC2 에 deploy.sh 을 생성한다.

regcred 인증 시간이 12시간 정도로 되어있어 할때 체크를 해서 진행해야하는데 개발서버이므로 다시 등록해주는 방식으로 진행했다.

```bash
#!/bin/bash

echo "k3s 권한 부여"
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

echo "login ecr"
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin ${ECR주소를 입력한다.}

echo "delete regcred"
kubectl delete secret regcred

echo "create regcred"
kubectl create secret generic regcred --from-file=.dockerconfigjson=/home/ubuntu/.docker/config.json --type=kubernetes.io/dockerconfigjson

echo "restart deployment"
kubectl rollout restart deployment/my-be # 본인의 Deployment 를 입력해 재실행
```
