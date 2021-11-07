➜  k8s-SingleMaster-18.9_9_w_auto-compl git:(main) ✗ vagrant halt

## create deployment

[root@m-k8s 4.3.4]# kubectl create deploy failure1 --image=multistage-img
deployment.apps/failure1 created



[root@m-k8s 4.3.4]# kubectl get pods -w
NAME                        READY   STATUS              RESTARTS   AGE
failure1-6dc55db9d4-5mcxr   0/1     ContainerCreating   0          7s
failure1-6dc55db9d4-5mcxr   0/1     ErrImagePull        0          8s
failure1-6dc55db9d4-5mcxr   0/1     ErrImagePull        0          9s
failure1-6dc55db9d4-5mcxr   0/1     ImagePullBackOff    0          9s



이유 이미지가 도커 허브에 없어서



[root@m-k8s 4.3.4]# kubectl create deploy failure2 --dry-run=client -o yaml --image=multistage-img > failure2.yaml



failure2.yaml

:       imagePullPolicy: Never 추가 = 외부에서 이미지를 가져오지 않고 호스트에 존재하는 이미지 사용

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: failure2
  name: failure2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: failure2
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: failure2
    spec:
      containers:
      - image: multistage-img
        imagePullPolicy: Never
        name: multistage-img
        resources: {}
status: {}
```



여전히 안됨

#### worker03에서 도커파일 빌드

docker build -t multistage-img-w3:latest .



#### 다시 마스터

[root@m-k8s 4.3.4]# sed -i 's/replicas: 1/replicas: 3/' success1.yaml
[root@m-k8s 4.3.4]# sed -i 's/failure2/success1/' success1.yaml



```yaml
[root@m-k8s 4.3.4]# cat success1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: success1
  name: success1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: success1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: success1
    spec:
      containers:
      - image: multistage-img
        name: multistage-img
        resources: {}
status: {}
```



3번만 가능함. 이유 이미지가 거기 있어서





# 레지스트리

[root@m-k8s 4.4.2]# ls
create-registry.sh  remover.sh  tls.csr



```sh
[root@m-k8s 4.4.2]# cat remover.sh
인증서를 만들어 배포한뒤 레지스트리를 구동하는


#!/usr/bin/env bash
certs=/etc/docker/certs.d/192.168.1.10:8443
mkdir /registry-image
mkdir /etc/docker/certs
mkdir -p $certs
openssl req -x509 -config $(dirname "$0")/tls.csr -nodes -newkey rsa:4096 \
-keyout tls.key -out tls.crt -days 365 -extensions v3_req

yum install sshpass -y
for i in {1..3}
  do
    sshpass -p vagrant ssh -o StrictHostKeyChecking=no root@192.168.1.10$i mkdir -p $certs
    sshpass -p vagrant scp tls.crt 192.168.1.10$i:$certs
  done

cp tls.crt $certs
mv tls.* /etc/docker/certs

docker run -d \
  --restart=always \
  --name registry \
  -v /etc/docker/certs:/docker-in-certs:ro \
  -v /registry-image:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/docker-in-certs/tls.crt \
  -e REGISTRY_HTTP_TLS_KEY=/docker-in-certs/tls.key \
  -p 8443:443 \
  registry:2




[root@m-k8s 4.4.2]# cat tls.csr
: 인증서를 만들때 사용하는 


[req]
distinguished_name = private_registry_cert_req
x509_extensions = v3_req
prompt = no

[private_registry_cert_req]
C = KR
ST = SEOUL
L = SEOUL
O = gilbut
OU = Book_k8sInfra
CN = 192.168.1.10

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.0 = m-k8s
IP.0 = 192.168.1.10






[root@m-k8s 4.4.2]# cat tls.csr
: 인증에 문제가 생겼을때 모든 파일을 지우는



[req]
distinguished_name = private_registry_cert_req
x509_extensions = v3_req
prompt = no

[private_registry_cert_req]
C = KR
ST = SEOUL
L = SEOUL
O = gilbut
OU = Book_k8sInfra
CN = 192.168.1.10

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.0 = m-k8s
IP.0 = 192.168.1.10
[root@m-k8s 4.4.2]# cat remover.sh
#!/usr/bin/env bash
certs=/etc/docker/certs.d/192.168.1.10:8443
rm -rf /registry-image
rm -rf /etc/docker/certs
rm -rf $certs

yum -y install sshpass
for i in {1..3}
  do
    sshpass -p vagrant ssh -o StrictHostKeyChecking=no root@192.168.1.10$i rm -rf $certs
  done

yum remove sshpass -y
docker rm -f registry
docker rmi registry:2
```





#### 레지스트리를 생성하고

[root@m-k8s 4.4.2]# sh create-registry.sh



[root@m-k8s 4.4.2]# docker ps -f name=registry
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                             NAMES
b920473ceb02        registry:2          "/entrypoint.sh /etc…"   17 seconds ago      Up 17 seconds       5000/tcp, 0.0.0.0:8443->443/tcp   registry





#### 레지스트리에 이미지  푸시

[root@m-k8s 4.4.2]# docker tag multistage-img 192.168.1.10:8443/multistage-img
[root@m-k8s 4.4.2]# docker images 192.168.1.10:8443/multistage-img
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
192.168.1.10:8443/multistage-img   latest              fe57afa49923        31 minutes ago      148MB
[root@m-k8s 4.4.2]# docker push 192.168.1.10:8443/multistage-img





#### 이미지가 정상적으로 등록됬는지 확인 : curl {레지스트리 주소} /v2/_catalog 



[root@m-k8s 4.4.2]# curl https://192.168.1.10:8443/v2/_catalog -k
{"repositories":["multistage-img"]}



### 도커 이미지가 2개 다 같은 것 확인

[root@m-k8s 4.4.2]# docker images | grep multi
192.168.1.10:8443/multistage-img     latest              fe57afa49923        34 minutes ago      148MB
multistage-img-w3                    latest              fe57afa49923        34 minutes ago

[root@m-k8s 4.4.2]# docker rmi -f fe57afa49923







## 직접 만든 이미지로 컨테이너 구동



[root@m-k8s 4.4.3]# mv /root/_Book_k8sInfra/ch4/4.3.4/success1.yaml success2.yaml
[root@m-k8s 4.4.3]# ls
audit-trail  echo-hname  echo-ip  success2.yaml





vi success2.yaml

      containers:
      - name: multistage
        image: 192.168.1.10:8443/multistage-img
        resources: {}

[root@m-k8s 4.4.3]# sed -i 's/success1/success2/' success2.yaml





[root@m-k8s 4.4.3]# kubectl apply -f success2.yaml
deployment.apps/success2 created

[root@m-k8s 4.4.3]# kubectl get pods -o wide
NAME                       READY   STATUS    RESTARTS   AGE   IP               NODE     NOMINATED NODE   READINESS GATES
success2-7565db55c-89q98   1/1     Running   0          46s   172.16.221.130   w1-k8s   <none>           <none>
success2-7565db55c-hm85d   1/1     Running   0          46s   172.16.103.133   w2-k8s   <none>           <none>
success2-7565db55c-t4kjk   1/1     Running   0          46s   172.16.132.3     w3-k8s   <none>           <none>





[root@m-k8s 4.4.3]# curl 172.16.221.130
src: 172.16.171.64 / dest: 172.16.221.130

[root@m-k8s 4.4.3]# curl 172.16.103.133
src: 172.16.171.64 / dest: 172.16.103.133

[root@m-k8s 4.4.3]# curl 172.16.132.3
src: 172.16.171.64 / dest: 172.16.132.3







```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: aws-load-balancer-controller
  namespace: kube-system
  labels:
   app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller

  annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::23423234:role/AmazonEKSLoadBalancerControllerRole
```

