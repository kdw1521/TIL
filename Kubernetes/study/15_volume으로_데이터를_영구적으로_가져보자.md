# Volume을 사용해 데이터를 영구적으로 관리하자.

k8s 는 프로그램에 기능이 추가되면 기존 파드에서 변경된 부분을 수정하지 않고, 새로운 파드를 만들어 통째로 갈아끼우는 방식으로 교체한다. (이게 효율적이라고 판단했음.)  
이런 특징 때문에 기존 파드 내부에 있던 데이터도 같이 삭제된다. 만일 이게 DB 를 실행하는 파드였다면 저장된 데이터가 모두 삭제된다.  
따라서, 내부 저장 데이터가 삭제되면 안되는 경우에 volume 을 사용한다.

volume 의 종류

- 로컬 볼륨
  - 파드 내부 공간 일부를 볼륨으로 활용하는 방식. 이방식은 파드가 삭제되면 즉시 데이터도 삭제되므로 실제로는 사용하지 않는다.
- 퍼시스턴트 볼륨 (Persistent Volume, PV)
  - 파드 외부의 공간 일부를 볼륨으로 활용하는 방식. 파드 삭제 여부와는 상관없이 데이터를 영구적으로 사용가능. 현업에서 많이 쓰임.
  - 퍼시스턴스 볼륨 클레임(Persistent Volume Claim, PVC)
    - 실제로 파드가 PV에 직접 연결할 수 없다. PVC라는 중개자가 있어야한다. 이 중개자를 통해 통신한다.

# 준비

- mysql-secret.yaml (Secret)
- mysql-config.yaml (ConfigMap)
- mysql-deployment.yaml (Deployment)
- mysql-service.yaml (Service)

1. mysql-secret.yaml 작성

```yaml
apiVersion: v1
kind: Secret

metadata:
  name: mysql-secret

stringData:
  mysql-password: 123qwe
```

2. mysql-config.yaml 작성

```yaml
apiVersion: v1
kind: ConfigMap

metadata:
  name: mysql-config

data:
  mysql-database: kub-practice
```

3. mysql-deployment.yaml 작성

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: mysql-deployment

spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql-db

  template:
    metadata:
      labels:
        name: mysql-db
    spec:
      containers:
        - name: mysql-container
          image: mysql
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql-password
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: mysql-config
                  key: mysql-database
```

4. mysql-service.yaml 작성

```yaml
apiVersion: v1
kind: Service

metadata:
  name: mysql-service

spec:
  type: NodePort
  selector:
    app: mysql-db
  ports:
    - protocol: TCP
      nodePort: 30002
      port: 3306
      targetPort: 3306
```

5. DB 커넥션 GUI 툴로 확인해본다.

- port: 30002
- user: root
- password: 123qwe

6. 볼륨을 사용하지 않았으므로 테스트 해본다.

- test 스키마를 생성하고 rollout 해본다.

```bash
> kubectl rollout restart deployment mysql-deployment
```

이렇게 하면 test 스키마는 지워지게 된다.
