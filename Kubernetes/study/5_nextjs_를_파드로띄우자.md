# NextJS 를 파드로 조작해보자.

NextJS 이미지를 만든 후 파드로 조작해본다.

# 준비

- NextJS
- Dockerfile

# 구현

1. Dockerfile 작성

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY . .

RUN npm install

RUN npm run build

EXPOSE 3000

ENTRYPOINT [ "npm", "run", "start" ]
```

2. next-pod.yaml 작성

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: next-pod

spec:
  containers:
    - name: next-container
      image: next-server
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 3000
```

3. 띄우고 확인 및 삭제는 나머지 3,4 와 동일하다.
