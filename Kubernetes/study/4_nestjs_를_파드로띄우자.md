# NestJS 를 파드로 조작해보자.

NestJS 를 이미지를 만든후 파드로 조작해본다.

# 준비

- NestJS (endpoint 아무거나 한개가 있는)
- Dockerfile
- nest-pod.yaml

# 구현

1. Dockerfile 작성

```dockerfile
FROM node

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

EXPOSE 3000

ENTRYPOINT [ "node", "dist/main.js" ]
```

기본적인 도커파일을 작성해준다.

2. nest-pod.yaml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nest-pod

spec:
  containers:
    - name: nest-container
      image: nest-server
      imagePullPolicy: IfNotPresent
```

3. 파드를 띄우고 확인한다.

```bash
> kubectl apply -f nest-pod.yaml
pod/nest-pod created

> kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
nest-pod   1/1     Running   0          21s
```

4. 접속을 해본다.

- 파드에 접속해서 확인

```bash
로컬> kubectl exec -it nest-pod -- bash
nest-pod> curl localhost:3000
Hello World!
```

- 포트 포워딩으로 확인

```bash
> kubectl port-forward nest-pod 3000:3000
```

4. 파드를 지운다.

```bash
> kubectl delete pod nest-pod
```
