# k8s 자동복구 기능
k8s 자동복구 기능이 있는데 이것을 확인해본다.

# 준비
- 여러개의 파드

# 구현
1. 컨테이너 한개를 죽여본다.
```bash
> docker kill {컨테이너 아이디}
```

2. 파드가 죽었는지 확인한다.
3. 예시
나는 5개의 파드중 한개를 죽여 확인해보았다.
```bash
> kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nest-deployment-84799f9fdb-dw7xr   1/1     Running   0          39m
nest-deployment-84799f9fdb-j6whq   1/1     Running   0          6s
nest-deployment-84799f9fdb-r6tpl   1/1     Running   0          6s
nest-deployment-84799f9fdb-v8fvt   1/1     Running   0          39m
nest-deployment-84799f9fdb-wjkc2   1/1     Running   0          39m

> docker ps    
CONTAINER ID   IMAGE          COMMAND               CREATED          STATUS          PORTS     NAMES
7f341e8d60df   3ccd8508d96a   "node dist/main.js"   16 seconds ago   Up 15 seconds             k8s_nest-conntainer_nest-deployment-84799f9fdb-r6tpl_default_41fa2774-b981-42fa-a5b5-a3d40aafeb16_0
1d426648c6a6   3ccd8508d96a   "node dist/main.js"   16 seconds ago   Up 15 seconds             k8s_nest-conntainer_nest-deployment-84799f9fdb-j6whq_default_31179fea-95a4-475f-9d32-351a2aac3900_0
2a1ffe1f6e5a   3ccd8508d96a   "node dist/main.js"   40 minutes ago   Up 40 minutes             k8s_nest-conntainer_nest-deployment-84799f9fdb-wjkc2_default_aad4e589-0bff-40b5-83e1-e8983609ecc9_0
c554200194d8   3ccd8508d96a   "node dist/main.js"   40 minutes ago   Up 40 minutes             k8s_nest-conntainer_nest-deployment-84799f9fdb-v8fvt_default_ce12a375-e097-4842-9b62-6ae66b527e6e_0
5fd1c21a5b08   3ccd8508d96a   "node dist/main.js"   40 minutes ago   Up 40 minutes             k8s_nest-conntainer_nest-deployment-84799f9fdb-dw7xr_default_60d329cc-0a42-4e76-97e9-5bc183b98d37_0

> docker kill 7f341e8d60df

> kubectl get pods
NAME                               READY   STATUS    RESTARTS     AGE
nest-deployment-84799f9fdb-dw7xr   1/1     Running   0            40m
nest-deployment-84799f9fdb-j6whq   1/1     Running   0            49s
nest-deployment-84799f9fdb-r6tpl   1/1     Running   1 (6s ago)   49s
nest-deployment-84799f9fdb-v8fvt   1/1     Running   0            40m
nest-deployment-84799f9fdb-wjkc2   1/1     Running   0            40m
```

이렇게 restarts 에 표시가 된다.