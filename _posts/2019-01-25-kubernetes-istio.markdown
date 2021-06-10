---
layout	: posts
title	: Kubernetes 학습(4/4) (Isito 구성)
summary	: Kubernetes
data	: 2019-01-25 08:09:36 +0900
updated	: 2019-01-25 08:09:36 +0900
comment	: true
categories: Kubernetes
tags:
  - Kubernetes
  - Istio 
  - MetalLB
---

* TOC
{:toc}

## Study with Kubernetes(4)

### 학습 일기
2018년도 Kubernetes 1.08 버전을 학습 하기 위해서 구성 및 테스트 한 내용을 정리 하였습니다. 해당 문서는 Kubernetes에 해당 문서는 Kubernetes에 Istio 구성을 위한 개념과 구성시 필요한 사항을 정리 하였습니다.

### Jenkins 설치 및 설정
istio 아키텍처에 간단한 설명

각각 데이터 플레인, 컨트롤 플레인으로 역할을 나누며, 데이터 플레인은 Envoy(Proxy)를 사이드카 형식으로 컨테이너 옆에 붙여서 배포하여 서비스로 들어오고 나가는 트래픽을 Envoy를 통하여 제어한다.

컨트롤 플레인은 Pilot, Mixer, Citadel 3개의 모듈로 구성 되어있으며, Pilot은 서비스들에 대한 End-Point 정보를 저장하고 이데이터를 가지고 Envoy는 Endpoint를 찾아 트래픽을 보낼수 있다.

  - Envoy: 각서비스 별로 Proxy를 제공하여 모듈들이 호출 하는 기능을 전달하는 역할

  - Mixer: 각종 모니터링 지표(Metric) 수집 및 접근 제어, 정책 관리를 담당하는 모듈

  - Pilot: envoy에 대한 설정 관리를 하는 역할 하는 모듈 - 서비스들의 End-Point 주소를 저장하여 Envoy에서 트래픽을 보낼수 있도록 Service Discovery 기능을 제공

  - Citadel: 사용자 인증(Authentication)과 인가(Authorization)등 보안에 관련된 기능을 담당하는 모듈(TLS 사용가능)

  - Galley: 사용자가 지정한 istio 설정을 받아서 각각 모듈에서 설정할수 있는 유효한 구성 설정으로 변경해주는 모듈

![Istio](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-4.png)

### Istio 설치
기본적으로 애플리케이션이 셋팅이 되었으면 Istio 를 설치 해서 Service Mash를 테스트 해보자

설치 방법은 2가지 방법이 있으며 지금 문서에서는 helm을 통하여 Istio를 설치할 것이다.

#### Istio-system namespace 생성
````
kubectl create namespace istio-system
````
  
#### Istio Custom Resource Definitions (CRDs)  설치 작업 (정의된 커스텀 리소스를 호출하기 위한 CRDs를 설치 해야한다.))
````
helm template install/kubernetes/helm/istio-init --name istio-init --namespace istio-system | kubectl apply -f -
````
  
#### 설치후 확인 방법 (53개의 crds가 설치되었는지 확인해야한다.)

````
kubectl get crds | grep 'istio.io\|certmanager.k8s.io' | wc -l

53
````

#### Istio 설치(helm을 사용하여 설치)법
````
helm install install/kubernetes/helm/istio --name istio --namespace istio-system \
    --values install/kubernetes/helm/istio/values-istio-demo.yaml
````

설치 완료시 아래와 같이 service가 뜬것을 확인이 가능하다.
````
[root@kube-master1 istio-1.1.1]# kubectl get svc -n istio-system
NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
grafana ClusterIP 10.233.0.132 <none> 3000/TCP 2m54s
istio-citadel ClusterIP 10.233.0.188 <none> 8060/TCP,15014/TCP 2m53s
istio-egressgateway ClusterIP 10.233.22.0 <none> 80/TCP,443/TCP,15443/TCP 2m54s
istio-galley ClusterIP 10.233.10.188 <none> 443/TCP,15014/TCP,9901/TCP 2m54s
istio-ingressgateway LoadBalancer 10.233.55.135 <pending> 80:31380/TCP,443:31390/TCP,31400:31400/TCP,15029:30053/TCP,15030:30604/TCP,15031:31573/TCP,15032:32389/TCP,15443:30979/TCP,15020:30274/TCP 2m54s
istio-pilot ClusterIP 10.233.42.68 <none> 15010/TCP,15011/TCP,8080/TCP,15014/TCP 2m54s
istio-policy ClusterIP 10.233.9.78 <none> 9091/TCP,15004/TCP,15014/TCP 2m54s
istio-sidecar-injector ClusterIP 10.233.60.127 <none> 443/TCP 2m53s
istio-telemetry ClusterIP 10.233.44.97 <none> 9091/TCP,15004/TCP,15014/TCP,42422/TCP 2m54s
jaeger-agent ClusterIP None <none> 5775/UDP,6831/UDP,6832/UDP 2m53s
jaeger-collector ClusterIP 10.233.17.118 <none> 14267/TCP,14268/TCP 2m53s
jaeger-query ClusterIP 10.233.29.228 <none> 16686/TCP 2m53s
kiali ClusterIP 10.233.24.246 <none> 20001/TCP 2m54s
prometheus ClusterIP 10.233.60.179 <none> 9090/TCP 2m54s
tracing ClusterIP 10.233.28.115 <none> 80/TCP 2m53s
zipkin ClusterIP 10.233.11.135 <none> 9411/TCP
````

#### Istio-injection 라벨 활성화
기본적으로 Istio 는 pod에 Envoy를 Sidecar 패턴으로 올려 트랙픽을 컨트롤 하는 구조를 가진다. Istio에서 sidecar를 pod 생성시 자동으로 주입해주는 기능을 활성화 하기위해서는 Kubernetes에 namespace에서 istio-injection=enabled 라벨을 추가해줘야한다.

````
[root@kube-master1 istio-1.1.1]# kubectl label namespace mservice istio-injection=enabled
namespace/mservice labeled
[root@kube-master1 istio-1.1.1]# kubectl label namespace tservice istio-injection=enabled
namespace/tservice labeled
[root@kube-master1 istio-1.1.1]# kubectl label namespace wservice istio-injection=enabled
namespace/wservice labeled
[root@kube-master1 istio-1.1.1]# kubectl get ns --show-labels
NAME STATUS AGE LABELS
default Active 8d istio-injection=enabled
ingress-nginx Active 8d app.kubernetes.io/name=ingress-nginx,app.kubernetes.io/part-of=ingress-nginx
istio-system Active 14m <none>
jenkins Active 8d <none>
kube-public Active 8d <none>
kube-system Active 8d name=kube-system
mservice Active 5h8m istio-injection=enabled
nexus3 Active 7d2h <none>
tservice Active 2d1h istio-injection=enabled
wservice Active 8d istio-injection=enabled
````

#### MetalLB
일반적으로 Istio를 사용할려면 클라우드 환경에서 제공해주는 LB가 필요하다.

istio는 Service Mash를 하기 위해서 Envoy를 사용하여 트래픽을 처리 하는대 이런 Envoy에서 트래픽을 처리하기 위해서는 상단의 Istio-ingressgateway를 통해 외부랑 연결되어야 할 필요가 있다. ignressgateway의 경우 외부랑 연결을 해주는 point가 필요하게 된다.

Kubernetes는 기본적으로 네트워크에 대한 표준 구현을 제공하지 않음(즉 기본적은 Cluster환경 내부에서 실행되는 애플리케이션을 액세스 할수 있지만 외부로 연결하기 위해서는 다른작업이 필요하다. ex)ingress, NordPort)

OpenShift에서는 위와같이 LB역할을 해주는것을 HA Proxy를 사용하여 처리하며 Kubernetes에는 아직 내부적으로 없기때문에 metalLB를 설치하여 처리할 예정이다.

여기서 간단히 Kubernetes 네트워크를 설명하지면 Kubernetes에서는 부하 분산 및 이중화 때문에 Service라는 단위로 여러 Pods를 한덩어리로 취급이 가능하다.(즉 아무리 Auto-scaling in, out이 발생하더라도 Service를 통해서 트래픽이 전달되기때문에 부하 분산이나 이중화가 가능)

Service는 Cluster IP 및 Load Balancer등의 종류가 있고 그중에서 Load Balancer 타입은 외부 통신 부하 분산용으로 사용 이 가능하다. 그러므로 Load Balancer 타입으로 설정된 경우 pod에 부하 분산을 해준다. 

클러스터 외부에서 애플리케이션을 액세스할려면 Load Balancer타입의 Service에 External IP가 할당 되어 있어야한다. 이러한 External IP를 할당 할수있는 LB가 필요하기때문에 일반적인 베어메탈 환경에서는 MetalLB나 다른 LB역할을 할수 있는 것을 설치할 필요가 있다.

기본적으로 Cloud 환경에서 제공 되는 LB를 사용했을 경우 네트워크 구조 이다.

![MetalLB](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-5.png)

MetalLB에서 External IP 할당 및 경로 정보를 갱신하는 순서

![순서](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-6.png)

  

외부에서 접근이 있는 경우 Router와 ipvs라 경로 정보를 가지고 연결을 컨트롤

![컨트롤](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-7.png)


#### MetalLB 설치
````
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: my-ip-space
      protocol: layer2
      addresses:
      - 192.168.23.220-192.168.23.223
````

##### ConfigMap 등록 - IP 대역 및 protocal 설정
````
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: my-ip-space
      protocol: layer2
      addresses:
      - 192.168.23.220-192.168.23.223
````
위와 같이 Configmap을 등록하면 IP pool에 등록된 IP에서(192.168.23.220 - 192.168.23.223) External IP할당 요청이 올시에 설정을 진행

위모드는 BGP 모드가 아닌 L2 layer 모드로 설정  


#### Gateway 및 virtualservice 설정
기본적으로 istio는 kubernetes에서 제공해주는 ingress와 service를 사용하지한고 enovy를 통하여 트래픽 제어를 하기 때문에 enovy에서 트래픽을 제어할 수 있도록 각 애플리케이션에 맞게 gateway와 virtualservice를 등록해야한다.

````
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: wserviceinfo
  namespace: wservice
spec:
  gateways:
  - wservice-gateway
  hosts:
  - wservice.app.paas-test.com
  http:
  - match:
    - uri:
        exact: /
    - uri:
        exact: /getLog.jsp
    - uri:
        exact: /test/serverLog
    route:
    - destination:
        host: wservice
        port:
          number: 80
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: tserviceinfo
  namespace: tservice
spec:
  gateways:
  - tservice-gateway
  hosts:
  - tservice.app.paas-test.com
  http:
  - match:
    - uri:
        exact: /
    - uri:
        exact: /test
    - uri:
        exact: /test/accessLog
    route:
    - destination:
        host: tservice
        port:
          number: 80
 
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: mserviceinfo
  namespace: mservice
spec:
  gateways:
  - mservice-gateway
  hosts:
  - mservice.app.paas-test.com
  http:
  - match:
    - uri:
        exact: /
    - uri:
        exact: /test
    - uri:
        exact: /test/serverLog
    - uri:
        exact: /test/test1
    - uri:
        exact: /test/test2
    - uri:
        exact: /test/http
    route:
    - destination:
        host: mservice
        port:
          number: 80
 
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: mservice-gateway
  namespace: mservice
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - mservice.app.paas-test.com
    port:
      name: http
      number: 80
      protocol: HTTP
 
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: tservice-gateway
  namespace: tservice
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - tservice.app.paas-test.com
    port:
      name: http
      number: 80
      protocol: HTTP
 
---
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: wservice-gateway
  namespace: wservice
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - wservice.app.paas-test.com
    port:
      name: http
      number: 80
      protocol: HTTP
````


### Grafana 연결
일반적으로 istio 셋팅시에 grafana 대시보드는 외부에서 노출 되지 않도록 되어있다.

외부에 노출이 안될시 확인 방법이 3가지 있는대

1. localhost에 3000번 포트를 포워딩 해준다.
  -  kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].
  metadata.name}') 3000:3000 & 

  - 위 명령어를 마스터 서버에서 쳐주면 해당 마스터 서버에서 로컬로 3000번이 포트포워딩 된다. (주의 할점은 마스터 서버에서만 접근이 가능하다.)

2. nginx-ingress를 통한 ingress 서비스를 등록해준다.

3. metalLB를 통한 External IP를 할당해준다.

이 문서에서 사용할 방법은 간단하게 metalLB를 통한 external IP를 할당하야 grafana에 접근 할 예정이다.

#### istio-system 프로젝트에서 grafana 서비스의 type을 LoadBalancer로 변경
````
#kubectl edit svc grafana -n istio-system
 
 
spec:
  clusterIP: 10.233.27.48
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 30194
    port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: grafana
  sessionAffinity: None
  type: LoadBalancer     <---- 해당부분을 변경
````

#### 확인 절차
1. kubectl 명령어나 웹UI에서 externelIP가 할당되었는지 확인 한다.
![Info](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-8.png)

2. grafan에 접근
![Grafana](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-9.png)


### Jaeger 연결
일반적으로 istio 셋팅시에 대시보드는 외부에서 노출 되지 않도록 되어있다.

외부에 노출이 안될시 확인 방법이 3가지 있는대

1. localhost에 3000번 포트를 포워딩 해준다.

    - kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0]. metadata.name}') 16686:16686 &

    - 위 명령어를 마스터 서버에서 쳐주면 해당 마스터 서버에서 로컬로 3000번이 포트포워딩 된다. (주의 할점은 마스터 서버에서만 접근이 가능하다.)

2. nginx-ingress를 통한 ingress 서비스를 등록해준다.

3. metalLB를 통한 External IP를 할당해준다.

이 문서에서 사용할 방법은 간단하게 metalLB를 통한 external IP를 할당하야 grafana에 접근 할 예정이다.

#### istio-system 프로젝트에서 grafana 서비스의 type을 LoadBalancer로 변경
````
#kubectl edit svc tracing -n istio-system
 
 
spec:
  clusterIP: 10.233.27.49
  externalTrafficPolicy: Cluster
  ports:
  - name: http
    nodePort: 30194
    port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    app: grafana
  sessionAffinity: None
  type: LoadBalancer     <---- 해당부분을 변경
````

#### Jaeger 접근 (Trace 값 확인 가능)
![Jaeger](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-10.png)
Trace 값은 호출에 따라 틀리다.


#### Trace 값 확인(wservice → mservice 호출)
첫번째로는 ingress-gateway를 통하여 wservice를 호출하고 wservice는 순차적으로 mservice는 mservice에서 tservice를 호출 하는 방식이다.
![Trace](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-11.png)

#### 현재 애플리케이션 DAG 확인(Directed acyclic graph)
![Trace](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-12.png)

## 결론
이처럼 MSA에 istio(Service Mesh) 사용하여 Config 레벨로 Service mesh를 하여 모니터링 및 트래픽 조절 등 여러가지를 가능하다는것을 볼수 있지만 Ingress 및 Egress 정책을 정확하게 수립할 필요가 있으며, 트래픽 모니터링 및 설정 자체에 아직 검증 되지 않는 부분이 많이 때문에 사용은 시기 상조로 보임

