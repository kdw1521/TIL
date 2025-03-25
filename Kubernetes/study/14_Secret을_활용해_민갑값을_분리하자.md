# 민감정보들을 Secret 으로 관리해보자.

ConfigMap 에는 노출되도 상관없는 값들을 관리하고 민감정보들은 Secret으로 관리한다.

# 준비

- 백엔드 프로젝트 이미지
- deployment.yaml
- config.yaml
- secret.yaml

# 구현

1. nest-secret.yaml 작성

```yaml
apiVersion: v1
kind: Secret

metadata:
  name: nest-secret

stringData:
  PASSWORD: wando-password-test
```

2. nest-config.yaml 수정

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: nest-config

data:
  NAME: wando
  # PASSWORD: pass 이부분 삭제
```

3. nest-deployment.yaml 수정

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
                  key: NAME_WANDO
            - name: PASSWORD
              valueFrom:
                secretKeyRef: # 이 부분이 secret 으로 변경
                  name: nest-secret # 이 부분도 secret의 name 으로 변경
                  key: PASSWORD_WANDO
```
