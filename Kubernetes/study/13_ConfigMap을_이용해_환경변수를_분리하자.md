# ConfigMap을 이용해 환경변수를 분리해보자.

Deployment에 환경변수를 등록하면 환경별로 서버를 실행할때 불편하다. 이때, ConfigMap을 이용해 환경변수를 분리한다.

# 준비

- 백엔드 프로젝트 (환경변수를 출력하는 endpoint 한개를 가진)
- deployment.yaml
- config.yaml

# 구현

NestJS 로 진행을 한다.

1. nest-config.yaml 작성

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: nest-config

data: # key value 쌍으로 입력해준다.
  NAME: wando
  PASSWORD: 1234
```

2. nest-deployment.yaml 작성

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
      labels:
        app: be-app
    spec:
      containers:
        - name: nest-container
          image: nest-server:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          env:
            - name: NAME
              valueFrom:
                configMapKeyRef:
                  name: nest-config
                  key: NAME # config에 있는 key
            - name: PASSWORD
              valueFrom:
                configMapKeyRef:
                  name: nest-config
                  key: PASSWORD # config에 있는 key
```

3. configmap 실행 후 deployment 재실행 및 확인

configmap 실행 & deployment 재실행

```bash
> kubectl apply -f nest-config.yaml
> kubectl apply -f nest-deployment.yaml
```

4. 로컬 브라우저에서 확인
   service에 nodePort를 30000으로 해두었고 NestJS 에 기본 endpoint에 해당 환경변수를 출력하게 해두어 브라우저로 확인한다.
