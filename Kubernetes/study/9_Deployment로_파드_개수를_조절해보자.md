# Deployment 로 파드 개수를 조절해보자.
트래픽이 늘거나 줄었을때 파드 개수를 늘리거나 줄이거나 해야한다. 이때 할수 있는 방법을 적용해본다.

# 준비
- 기존 deployment.yaml 파일

# 구현
1. 기존 파일에서 spec.replicas 의 개수를 변경해준다.
2. kubectl apply -f nest-deployment.yaml
3. 예시
```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nest-deployment

spec:
  replicas: 3 # 이 부분의 개수를 변경해준다.
  selector:
    matchLabels:
      app: be-app

  template:
    metadata:
      labels:
        app: be-app
    spec:
      containers:
        - name: nest-conntainer
          image: nest-server
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
```

이렇게 히면 자동으로 개수가 변경이 되어 적용된다.