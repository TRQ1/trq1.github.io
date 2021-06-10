---
layout	: posts
title	: Kubernetes 학습(1/4) (Install)
summary	: Kubernetes
data	: 2019-01-22 08:09:36 +0900
updated	: 2019-01-22 08:09:36 +0900
comment	: true
categories: Kubernetes
tags:
  - Kubernetes 
---

* TOC
{:toc}

## Study with Kubernetes(1)

### 학습 일기
2018년도 Kubernetes 1.08 버전을 학습 하기 위해서 구성 및 테스트 한 내용을 정리 하였습니다. 해당 문서는 Kubernetes를 하이퍼바이저에 셋팅한 구성 내용 입니다.


### 물리 구성
![Infra](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/Hardware.png)

### 쿠버네티스 기본 셋팅
Kubernetes를 구성하기위해 아래 와같이 [Kubespray](https://github.com/kubernetes-sigs/kubespray)를 사용하여 설치 하였습니다.


### 저장소 설정
Node 가 2개 이상이기 때문에 NFS 공유 볼륨을 이용하여 공유 저장소를 구성 하였습니다.

#### NFS 구성
Master1 서버에 NFS 서버를 설치
```
## Master1
/etc/exports 설정

/NFS/data 192.168.23.* (rw,no_all_squash,no_root_squash,sync)/NFS/data 10.*.*.* (rw,no_all_squash,no_root_squash,sync)
```

#### 공유 디렉토리 생성 
```
mkdir /NFS/data/jenkinsReg
mkdir  /NFS/data/dockerReg
chmod 777 /NFS/data/
```

#### hosts.ini 파일
```
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
kube-master1 ansible_host=192.168.23.170  # ip=10.3.0.1 etcd_member_name=etcd1
kube-master2 ansible_host=192.168.23.174  # ip=10.3.0.2 etcd_member_name=etcd2
kube-master3 ansible_host=192.168.23.175  # ip=10.3.0.3 etcd_member_name=etcd3
kube-node1 ansible_host=192.168.23.171  # ip=10.3.0.4 etcd_member_name=etcd4
kube-node2 ansible_host=192.168.23.172  # ip=10.3.0.5 etcd_member_name=etcd5
kube-node3 ansible_host=192.168.23.173  # ip=10.3.0.6 etcd_member_name=etcd6
 
# ## configure a bastion host if your nodes are not directly reachable
# bastion ansible_host=x.x.x.x ansible_user=some_user
 
[kube-master]
kube-master1
kube-master2
kube-master3
 
[etcd]
kube-master1
kube-master2
kube-master3
 
[kube-node]
kube-node1
kube-node2
kube-node3
 
[k8s-cluster:children]
kube-master
kube-node
```
 

### 인증 정보 추가

UI 대시보드를 사용하기 위해서는 cluster-admin 이라는 Role을 할당해야 한다.

#### UI 대시보드 사용을 위한 인증 정보 추가
```
cat > dashboard-admin-userAndRole.yaml <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
 
---
 
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
 
kubectl create -f dashboard-admin-userAndRole.yaml
```
 

#### 인증 Token 확인
```
 kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
 

대시보드 URL 확인

kubectl cluster-info
 
Kubernetes master is running at https://192.168.23.170:6443
coredns is running at https://192.168.23.170:6443/api/v1/namespaces/kube-system/services/coredns:dns/proxy
kubernetes-dashboard is running at https://192.168.23.170:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
KubeRegistry is running at https://192.168.23.170:6443/api/v1/namespaces/kube-system/services/registry:registry/proxy

```

#### 대시보드 접속
![Dashboard](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/kubedashboard.png) 

ex) Addon에서 Reigstry를 설치를 활성화시 pvc가 없어 pod가 sacle out 되지 않는 경우 아래와 같이 kube-system에 pv, pvc 추가
```
Docker-Registry Persistent Volume

apiVersion: v1
kind: PersistentVolume
metadata:
  name: registry-pv
spec:
  capacity:
    storage: 30Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 192.168.23.170
    path: "/NFS/data/dockerReg"
 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 30Gi
```
 

### DNS 설정

#### CoreDNS 에 외부 DNS 정보를 추가

CoreDNS 에 없는 정보에 대해서 찾을 수 있도록 외부 DNS를 등록한다.(192.168.23.2, 8.8.8.8 등)

ex) configmap edit시에 upstream과 proxy부분에 회사 DNS 및 외부 DNS를 등록한다.
```
kubectl edit configmap coredns -n kube-system
 
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          upstream 192.168.23.2 8.8.8.8
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        proxy . 192.168.23.2
        cache 30
        loop
        reload
        loadbalance
    }
kind: ConfigMap
```

### Docker Daemon 설정

#### Private Registry 정보 확인

```
kubectl describe svc registry -n kube-system
 
Name:              registry
Namespace:         kube-system
Labels:            addonmanager.kubernetes.io/mode=Reconcile
                   k8s-app=registry
                   kubernetes.io/cluster-service=true
                   kubernetes.io/name=KubeRegistry
Annotations:       kubectl.kubernetes.io/last-applied-configuration:
                     {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"addonmanager.kubernetes.io/mode":"Reconcile","k8s-app":"regist...
Selector:          k8s-app=registry
Type:              ClusterIP
IP:                10.233.19.81
Port:              registry  5000/TCP
TargetPort:        5000/TCP
Endpoints:         10.233.74.6:5000
Session Affinity:  None
Events:            <none>

```

#### Kubernetes에서 Docker 네트워크 활성화가 필요시

기본적으로 Kubernetes가 활성화 되면 kubernetes에서는 Docker network가 활성화를 하지 않게 한다.

즉 현재 node들에서 docker build를 하여도 docker image에서 외부로 나가는 망이 차단되기때문에 인터넷을 이용한 패키지 설치가 필요한 경우 네트워크에서 차단될수있다.

이것을 활성화 하기 위해서는 아래와 같이 iptables 추가 및 docker configuration을 변경해야한다.

하지만 기본적으로 보안상 맞아놓은 거기때문에 외부에서 docker image를 만들어서 가져오거나 외부 dockerhub에서 바로 받는것을 추천한다.

```
iptables -t nat -N DOCKER
iptables -t nat -A DOCKER -i docker0 -j RETURN
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
iptables -t nat -A OUTPUT ! -d 127.0.0.0/8 -m addrtype --dst-type LOCAL -j DOCKER
iptables -A FORWARD -i docker0 -o ens3 -j ACCEPT
iptables -A FORWARD -i ens3 -o docker0 -j ACCEPT
 
 
vi /etc/systemd/system/docker.service.d/docker-options.conf
--iptables=true 로 변경
```
 

#### Master 및 모든 Node 에 동일하게 설정

내부에서 Private Registry를 접근하기 위해서는 아래와 같이 insecure-registries 설정을 해야 한다.
```
[service].[namespace].svc.cluster.local (클러스터 내부에 접속시에 svc.cluster.local로 접근이 가능하다)

vi /etc/docker/daemon.json
{
  "insecure-registries": ["registry.kube-system.svc.cluster.local:5000"]
}
systemctl restart docker
```
 

#### Ingress Nginx Controller

사용자의 모든 요청은 Ingress Ngin Controller 를 통해서 각 Ingress 를 찾는다.

사용자가 애플리케이션까지 도달하기 위해서 아래와 같은 단계를 거친다.

1. 사용자 Browser (지정된 URL)

2. 외부의 DNS 를 통해 내부 VIP IP 획득

3. Master 서버의 API가 Haproxy VIP를 호출

4. Ingress Nginx Controller
    - 지정된 URL에 맞는 Ingress 탐색

5. Ingress )
    - Servie 와 맵핑
    - VirtualHost 설정과 같이 host 에 도메인을 설정한다.

6. Service
    - Pod 목록을 가지고 있다.(CodeDNS 이용)

7. Pod (Deploment)

#### Ingress Nginx Controller 설정
```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.20.0/deploy/mandatory.yaml

kubectl create -f mandatory.yaml
```
 
### 정상적으로 Running 인지 확인
kubectl get pods -n ingress-nginx
```
NAME                                        READY   STATUS    RESTARTS   AGE
default-http-backend-85b8b595f9-vqdcn       1/1     Running   0          48s
nginx-ingress-controller-866568b999-qd7gg   1/1     Running   0          48s
 
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.20.0/deploy/provider/baremetal/service-nodeport.yaml
kubectl create -f service-nodeport.yaml
 
## NodePort 추가(추가 하지 않을 경우 30000-32767 사이에서 랜덤하게 설정) kubectl create -f service-nodeport.yaml
root@kube-master1 ~]# kubectl get svc -n ingress-nginx
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
default-http-backend   ClusterIP   10.233.31.22   <none>        80/TCP                       103m
ingress-nginx          NodePort    10.233.18.37   <none>        80:32763/TCP,443:30951/TCP   102m

