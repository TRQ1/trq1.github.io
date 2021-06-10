---
layout	: posts
title	: Kubernetes 학습(2/4) (WAS 구성)
summary	: Kubernetes
data	: 2019-01-23 08:09:36 +0900
updated	: 2019-01-23 08:09:36 +0900
comment	: true
categories: Kubernetes
tags:
  - Kubernetes 
  - Wildfly
  - Tomcat
  - OpenJdk
---

* TOC
{:toc}

## Study with Kubernetes(2)

### 학습 일기
2018년도 Kubernetes 1.18 버전을 학습 하기 위해서 구성 및 테스트 한 내용을 정리 하였습니다. 해당 문서는 Kubernetes에 WAS를 셋팅한 내역입니다.


### Wildfly 16 설치
````
kubectl create namespace service
 
 
cat > deployment-wildfly16.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wservice
spec:
  selector:
    matchLabels:
      app: wservice
  replicas: 1
  template:
    metadata:
      labels:
        app: wservice
    spec:
      containers:
      - name: wservice
        image: jboss/wildfly
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
EOF
 
kubectl create -f deployment-wildfly16.yaml -n service
 
cat > service-wildfly16.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: wservice
spec:
  ports:
    - name: "http"
      port: 80
      protocol: TCP
      targetPort: 8080
  selector:
    app: wservice
EOF
 
kubectl create -f service-wildfly16.yaml -n service
 
cat > ingress-wildfly16.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: wservice
spec:
  rules:
  - host: wservice.app.paas-test.com
    http:
      paths:
      - backend:
          serviceName: wservice
          servicePort: 8080
        path: /
EOF
 
kubectl create -f ingress-wildfly16.yaml -n service
````

### Tomcat 설정
````
cat > deployment-tomcat.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tservice
spec:
  selector:
    matchLabels:
      app: tservice
  replicas: 1
  template:
    metadata:
      labels:
        app: tservice
    spec:
      containers:
      - name: tservice
        image: tomcat:latest
        ports:
        - containerPort: 8080
EOF
 
kubectl create -f deployment-tomcat.yaml -n service
 
cat > service-tomcat.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: tservice
spec:
  ports:
    - name: "http"
      port: 80
      protocol: TCP
      targetPort: 8080
  selector:
    app: tservice
EOF
 
kubectl create -f service-tomcat.yaml -n service
 
cat > ingress-tomcat.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: tservice
spec:
  rules:
  - host: tservice.app.paas-test.com
    http:
      paths:
      - backend:
          serviceName: tservice
          servicePort: 8080
        path: /
EOF
 
kubectl create -f ingress-tomcat.yaml -n service
````

### SpringBoot 설정
````
kubectl create namespace mservice
 
cat > deployment-openjdk.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mservice
spec:
  selector:
    matchLabels:
      app: mservice
  replicas: 1
  template:
    metadata:
      labels:
        app: mservice
    spec:
      containers:
      - name: mservice
        image: openjdk:8u201-jdk-alpine3.9
        ports:
        - containerPort: 8080
EOF
 
kubectl create -f deployment-openjdk.yaml -n service
 
cat > service-openjdk.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: mservice
spec:
  ports:
    - name: "http"
      port: 80
      protocol: TCP
      targetPort: 8080
  selector:
    app: mservice
EOF
 
kubectl create -f service-openjdk.yaml -n service
 
cat > ingress-openjdk.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: mservice
spec:
  rules:
  - host: mservice.app.paas-test.com
    http:
      paths:
      - backend:
          serviceName: mservice
          servicePort: 8080
        path: /
EOF
 
kubectl create -f ingress-openjdk.yaml -n service
````