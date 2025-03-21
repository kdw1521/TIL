# Spring Boot 를 파드로 조작해보자.
Spring Boot 를 이미지를 만든 뒤 파드로 띄워 조작해본다.

# 준비
- Spring Boot 프로젝트 (endpoint 아무거나 한개 준비)
- Dockerfile
- spring-server.yaml

# 구현
1. Dockerfile 작성
```dockerfile
FROM openjdk:17-jdk

COPY build/libs/*SNAPSHOT.jar app.jar

ENTRYPOINT [ "java", "-jar", "/app.jar" ]
```
기본적인 도커파일을 작성해준다.

2. spring-server.yaml
```yaml
apiVersion: v1
kind: Pod

metadata:
  name: spring-pod

spec:
  containers:
    - name: spring-container
      image: spring-server
      ports:
        - containerPort: 8080
      imagePullPolicy: IfNotPresent
```
imagePullPolicy 옵션을 지정하지 않으면 파드를 띄울떄 에러가 난다.  
이미지 풀 정책이 있는데 정리를 해본다.
1. Always - 로컬에서 이미지를 가져오지 않고, 무조건 레지스트리(= Dockerhub, ECR과 같은 원격 이미지 저장소)에서 가져온다. 
2. IfNotPresent - 로컬에서 이미지를 가져온다. 없다면 레지스트리에서 가져온다.
3. Never - 로컬에서만 이미지를 가져온다.  

이렇게 3가지 정책이 있는데
- 이미지의 태그가 `latest` 거나 `없는 경우`는 `default로 Always` 정책이 적용된다.
- 이미지 태그가 있다면 `IfNotPresent`이 default 로 적용된다.

3. 파드를 띄우고 확인한다.
```bash
> kubectl apply -f spring-server.yaml
> kubectl get pods
```

4. 접속을 해본다.
- 파드에 접속해서 endpoint 호출
```bash
로컬> kubectl exec -it spring-pod -- bash
spring-pod> curl localhost:8080
```

- 포트 포워딩으로 호출
```bash
> kubectl port-forward pod/spring-pod 12345:8080
```
로컬에서 localhost:12345를 호출해 확인한다.

5. 파드를 지운다.
```bash
> kubectl delete pod spring-pod
```