# Service를 이용해 백엔드 파드 여러개에 균등하게 요청을 보내보자.
7 번에서 Deployment를 이용해 백엔드를 여러개 띄웠다. 이 백엔드 여러개에 균등하게 요청을 보내야 의미가 있다. 이때 Service를 이용하여 작업해본다.

# 준비
- 백엔드 프로젝트 (with. Docker image)
- nest-deployment.yaml
- nest-service.yaml

# 구현
1. nest-service.yaml 작성
```yaml
apiVersion: v1
kind: Service

metadata:
  name: nest-service

spec:
  type: NodePort # 하단 참고 확인
  selector:
    app: be-app # deployment 에서 정의한것과 매칭
  ports:
    - protocol: TCP
      port: 3000 # k8s 내부에서 Service에 접속하기 위한 포트번호
      targetPort: 3000 # 매핑하기 위한 파드에 있는 포트
      nodePort: 30000 # 클라이언트가 요청보내는 서버의 포트
```

2. 띄우고 확인
```bash
> kubectl apply -f nest-service.yaml
service/nest-service created

> kubectl get service
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP          2d22h
nest-service   NodePort    10.98.221.123   <none>        3000:30000/TCP   8s
```

# 참고
Service는 type(종류)이 존재한다. 일단 3가지를 알아본다.
- NodePort: k8s 내부에서 해당 서비스에 접속하기 위한 포트를 열고 외부에서 접속 가능하도록 한다.
- ClusterIP: k8s 내부에서만 통신 할 수 있는 IP 주소를 부여. 외부에서는 요청할수 없다.
- LoadBalancer: 외부의 LB를 활용해 외부에서 접속할 수 있도록 연결한다.