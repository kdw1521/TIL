# Deployment 로 백엔드를 여러개 띄워보자.
기존에는 pod.yaml 에 --- 로 구분하여 name 을 다르게 주어 여러개를 띄울수가 있었는데 매우 번거롭고 관리하기 귀찮다. 그래서 Deployment 를 이용해 띄워본다.

# 준비
- 백엔드 프로젝트 (여기선 NestJS로 한다.)
- Dockerfile
- nest-deployment.yaml

# 구현
1. deployment.yaml 작성
```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: nest-deployment

# Deployment 세부정보
spec:
  replicas: 3 #  복제본 개수
  selector:
    matchLabels:
      app: be-app # 이런 이름을 가진 파드를 기반으로 띄운다. (여기선 하단의 template이 됨.)

  # 배포할 파드 정의  
  template:
    metadata:
      labels: # 카테고리로 이해
        app: be-app
    spec:
      containers:
        - name: nest-conntainer
          image: nest-server
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
```

2. 띄워보자.
```bash
> kubectl apply -f nest-deployment.yaml
deployment.apps/nest-deployment created
```

3. 확인해보자.
```bash
> kubectl get deployment
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
nest-deployment   3/3     3            3           23s

> kubectl get replicaset
NAME                         DESIRED   CURRENT   READY   AGE
nest-deployment-84799f9fdb   3         3         3       59s

> kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nest-deployment-84799f9fdb-dw7xr   1/1     Running   0          9s
nest-deployment-84799f9fdb-v8fvt   1/1     Running   0          9s
nest-deployment-84799f9fdb-wjkc2   1/1     Running   0          9s
```