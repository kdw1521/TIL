# 파드를 디버깅 해보자.

파드를 실행할때나 여러 이유로 에러가 발생할 수 있다. 이때, 디버깅을 통해 어디 부분이 문제인지 빠르게 찾아 해결할수 있도록 해본다.

# 준비

- nginx-pod.yaml

# 구현

1. 없는 이미지를 파드로 띄워보자.

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: nginx-pod

spec:
  containers:
    - name: nginx-container
      image: nginx:1.92.3
      ports:
        - containerPort: 80
```

1.92.3 이렇게 없는 tag를 추가하고 실행해 본다.

2. 실행 및 결과

```bash
> kubectl apply -f nginx-pod.yaml
pod/nginx-pod created

> kubectl get pods
nginx-pod    0/1     ErrImagePull   0          10s
```

ErrImagePull 이란 상태로 이미지 가져올때 문제가 생겼거니 생각은 되지만, 상세하게 분석해 해결이 필요하다.

3. describe

```bash
> kubectl describe pod nginx-pod
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  2m46s                default-scheduler  Successfully assigned default/nginx-pod to docker-desktop
  Normal   Pulling    74s (x4 over 2m46s)  kubelet            Pulling image "nginx:1.92.3"
  Warning  Failed     73s (x4 over 2m45s)  kubelet            Failed to pull image "nginx:1.92.3": Error response from daemon: failed to resolve reference "docker.io/library/nginx:1.92.3": docker.io/library/nginx:1.92.3: not found
  Warning  Failed     73s (x4 over 2m45s)  kubelet            Error: ErrImagePull
  Warning  Failed     46s (x6 over 2m44s)  kubelet            Error: ImagePullBackOff
  Normal   BackOff    34s (x7 over 2m44s)  kubelet            Back-off pulling image "nginx:1.92.3"
```

더 많은 내용이 있지만, Evnets 를 확인한다.

4. logs
   있는 버전으로 돌린뒤 파드를 띄운다. 그리고 logs 를 이용해 로그를 볼수 있다.

```bash
> kubectl logs nginx-pod
```
