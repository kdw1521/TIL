# k8s 를 통해 nginx 를 조작해보자.

nginx pod 를 yaml을 이용해 간단하게 실행하고 접속하고 지워본다.

# 준비

- k8s
- nginx-pod.yaml

# 구현

1. nginx-pod.yaml

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-pod

spec:
  containers:
    - name: nginx-container
      image: nginx
      ports: # 이건 명세서 같은 것이다. 실제로 띄울때는 아무런 영향을 미치지 않는다.
        - containerPort: 80
```

2. `kubectl apply -f nginx-pod.yaml` 로 실행
3. `kubectl get pods` 로 확인

하지만, 이렇게 해도 localhost:80 으로 접속 하면 접속이 되지않는다. 파드(파드내부는 전부 동일 네트워크)와 로컬머신의 네트워크가 분리되어있기 때문이다. (Docker 에서 컨테이너 내부/외부 분리된것과 유사)  
접근할수 있는 2가지 방법이 있는데,

1. 파드 내부로 들어가서 접근
2. 파드 내부 네트워크를 외부에서도 접속 할 수 있도로 포트 포워딩 활용

두가지 방법으로 접근해본다.

1. 파드 내부로 들어가서 접근

```bash
로컬머신> kubectl exec -it nginx-pod -- bash
파드내부> curl localhost:80
```

위처럼 접근(Docker 와 유사)해 요청을 날려보면 응답이 잘 온다.

2. 파드 내부 네트워크를 외부에서도 접속 할 수 있도로 포트 포워딩 활용

```bash
> sudo kubectl port-forward pod/nginx-pod 80:80
```

위와 같이 포트 포워딩을 통해 로컬머신과 연결해 확인이 가능하다. (맥은 80에 sudo 권한이 필요하다)

마지막으로 해당 파드를 지워본다.

```bash
> kubectl delete pod nginx-pod
```
