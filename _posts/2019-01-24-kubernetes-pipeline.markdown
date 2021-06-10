---
layout	: posts
title	: Kubernetes 학습(3/4) (Pipeline 구성)
summary	: Kubernetes
data	: 2019-01-24 08:09:36 +0900
updated	: 2019-01-24 08:09:36 +0900
comment	: true
categories: Kubernetes, Pipeline
tags:
  - kubernetes, Jenkins
---

* TOC
{:toc}

## Study with Kubernetes(3)

### 학습 일기
2018년도 Kubernetes 1.18 버전을 학습 하기 위해서 구성 및 테스트 한 내용을 정리 하였습니다. 해당 문서는 Kubernetes에 Jenkins를 활용하여 Pipeline을 구성 한 내용입니다.


### Jenkins 설치 및 설정
````
kubectl create namespace jenkins
 
 
cat > pv-pvc-jenkins.yaml <<EOF
kind: PersistentVolume
apiVersion: v1
metadata:
  name: jenkins-pv
  namespace: jenkins
spec:
  capacity:
    storage: 30Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.23.170
    path: /NFS/data/jenkinsReg
     
---
     
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
EOF
 
kubectl create -f pv-pvc-jenkins.yaml
 
cat > deployment-jenkins.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins
        image: jenkins/jenkins:2.150.3
        ports:
        - containerPort: 8080
        volumeMounts:
          - mountPath: /var/jenkins_home
            name: jenkins-data-volume
      volumes:
        - name: jenkins-data-volume
          persistentVolumeClaim:
            claimName: jenkins-pvc
EOF
     
kubectl create -f deployment-jenkins.yaml
 
cat > service-jenkins.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    app: jenkins
EOF
     
kubectl create -f service-jenkins.yaml
 
cat > ingress-jenkins.yaml <<EOF
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: jenkins
  namespace: jenkins
spec:
  rules:
  - host: jenkins.app.paas-test.com
    http:
      paths:
      - backend:
          serviceName: jenkins
          servicePort: 8080
        path: /
         
EOF
 
kubectl create -f ingress-jenkins.yaml
````

#### Jenkins Role 설정(Service account 설정)
````
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins-role-binding
  namespace: kube-public
  labels:
    app: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
````

#### Jenkins-JNLP 슬레이브 서비스 포트(빌드시 gradle-slave를 사용하여 build 하기 위해서 반드시 50000 포트가 열려있어야한다)
````
apiVersion: v1
kind: Service
metadata:
  name: jenkins-jnlp
  namespace: jenkins
spec:
  ports:
    - port: 50000
      targetPort: 50000
  selector:
    app: jenkins
````

#### Jenkins system:serviceaccount:default 권한주기
기본적으로 slave pod에 올라가는 namespace의 계정에서 사용에 필요한 권한이 있지 않는 경우 kubectl 명령어를 사용한다고 하더라도 권한이 없어 api를 날리지 못한다\

그렇기때문에 pipeline으로 자동화하여 배포를 하기 위해서는 그에 준한 권한을 줘야한다.

````
kubectl create rolebinding jenkin-perm --clusterrole=edit --user=system:serviceaccount:jenkins:default -n service
````

- 기본적으로 RBAC 참고시 아래 사이트를 참고해야한다.
https://kubernetes.io/docs/concepts/policy/pod-security-policy/

- 다른 방법으로는 수동으로 아래와 같이 Role 설정을 해야한다.  
아래의 권한은 Deployment 리소스에서 권한을 주기 위한 RBAC 설정이다.
````
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: wservice
  name: deployment-edit
rules:
- apiGroups: ["extensions", "apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
````

#### 초기 비밀번호 확인 방법
파드(Pod) 메뉴 -> Pod -> 로그 클릭으로 확인  
웹페이지 접속후 pod에서 경로 접속하여 패스워드 확인

### Kubernetes plugin을 사용하여 연결 작업

#### Jenkins와 연결하기 위해 Kubernetes 요구 내용 준비

1. 클러스터 서버 IP 확인  
```
kubectl config view | grep server | cut -f 2- -d ":" | tr -d " "
```
2. 클러스터에 접근가능한 Token 확인
  - admin 권한 확인시: kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

  - 일반유저 권한 확인시: kubectl describe secret $(kubectl get secrets | grep default | cut -f1 -d ' ') | grep -E '^token' | cut -f2 -d':' | tr -d '\t'    → 예제로 default 프로젝트에 접근가능한 token 생성
3. Kubernetes server certificate key 확인
```
cat .kube/config   →  certificate-authority-data 항목 확인
```
4. certificate-authority-data 값을 base64로 decode
```
export certificatedata='certificate-authority-data 값넣기'

echo $certificatedata | base64 --decode
```

#### Jenkins/Kubernetes 연결 작업
1. Jenkins 접속
2. Manage Jenkins → Configure system 이동
3. Cloud 항목 이동 → Add a new cloud 선택
4. 아래와 항목을 입력
  - Name: kubernetes
  - Kubernetes URL: 클러스터 서버 IP
  - Kubernetes server certificate key: 디코드된 certificate-authority-data
  - Kubernetes Namespace: Kubernetes 기본 Namespace 
  - Credential: Kubernetes 접근가능한 인증 → Add → Jenkins
  ![기존에 수집한 Secret정보와 ID 값 정의 후 추가](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-1.png)
  - Jenkins URL: 사용중인 Jenkins URL 입력

5. Jenkins slave tunnel 작업 (Service에서 Jenkins-jnlp 아이피를 확인후 아래와 같이 tunnel 설정을 해줘야한다.)
![Jenkins slave tunnel 작업](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-2.png)

#### Jenkins-Slave Pod 만들기(Custom)
```
FROM centos:7.5.1804
 
 
#Required Package
RUN \
        yum install -y java java-devel sudo skopeo unzip && \
        yum clean all \
        && \
        yum-config-manager --disable *
 
 
RUN yum install -y git
RUN curl -O -sSL https://download.docker.com/linux/centos/7/x86_64/edge/Packages/docker-ce-18.06.3.ce-3.el7.x86_64.rpm
RUN yum localinstall -y --nogpgcheck docker-ce-18.06.3.ce-3.el7.x86_64.rpm
RUN rm -f docker-ce-18.06.3.ce-3.el7.x86_64.rpm
 
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.13.5/bin/linux/amd64/kubectl
RUN chmod +x ./kubectl
RUN mv ./kubectl /usr/local/bin/kubectl
 
RUN curl -skL -o /tmp/gradle-bin.zip https://services.gradle.org/distributions/gradle-5.3.1-bin.zip && \
    mkdir -p /opt/gradle && \
    unzip -q /tmp/gradle-bin.zip -d /opt/gradle && \
    ln -sf /opt/gradle/gradle-5.3.1/bin/gradle /usr/local/bin/gradle
 
RUN chown -R 1001:0 /opt/gradle && \
    chmod -R g+rw /opt/gradle/gradle-5.3.1
 
 
# Define location of the Oracle JDK
ENV JAVA_HOME /usr/lib/jvm/java
# Define location of the Oracle JRE
ENV JRE_HOME /usr/lib/jvm/jre
# Defile localtion of the Gradle
ENV GRADLE_HOME=/opt/gradle/gradle-$GRADLE_VERSION/
 
## As depending on docker guid, you should change docker guid
RUN sed -i 's/docker:x:995:/docker:x:993:jenkins/g' /etc/group
 
# Download the Jenkins Slave JAR
RUN curl --create-dirs -sSLo /usr/share/jenkins/slave.jar https://repo.jenkins-ci.org/public/org/jenkins-ci/main/remoting/3.9/remoting-3.9.jar \
        && chmod 755 /usr/share/jenkins \
        && chmod 644 /usr/share/jenkins/slave.jar
 
# Download the Jenkins Slave StartUp Script
RUN curl --create-dirs -sSLo /usr/local/bin/jenkins-slave https://raw.githubusercontent.com/jenkinsci/docker-jnlp-slave/3.27-1/jenkins-slave \
        && chmod a+x /usr/local/bin/jenkins-slave
 
# Add a dedicated jenkins system user
RUN useradd --system --shell /bin/bash --create-home --home /home/jenkins jenkins
 
# This is actually a very dirty hack because it grants sudo privilieges to user `jenkins` without password!
#
# Unfortunately the CentOS installation needs some further adaptions to project specific needs which
# cannot (or shoudn't) be done on the public internet (e.g. modify /etc/hosts, add certificates to java keystore, ...).
#
# If there's a better way to customize the installation during runtime with root access, you're welcome to improve
# this Dockerfile or to describe the approach.
#
RUN echo "jenkins ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/jenkins
 
# Switch to user `jenkins`
USER jenkins
 
# Prepare the workspace for user `jenkins`
RUN mkdir -p /home/jenkins/.jenkins
VOLUME /home/jenkins/.jenkins
WORKDIR /home/jenkins
 
ENTRYPOINT ["jenkins-slave"]
````


### 테스트 애플리케이션 배포 절차
#### Jenkinspipeline 을 활용하여 배포
-  각각 Source상에 Dockerfile 및 Jenkins 파일을 넣음

아래의 5개 stage로 구성됨

- Jenkins : Project Build Now

- Gradle : WAR Clean Build

- Gradle : Docker Build

- Gradle : Private Registry Push(docker gradle plugin)

- Kube Master : Kube Deployment Image Change

아래와 같은 jenkinspipeline 생명주기 이다.

![Jenkinspipeline 생명주기](https://github.com/TRQ1/trq1.github.io/raw/master/_post_img/jenkins-kubernetes-2.png)
- 위 이미지는 이해를 위해 블로그에서 가져옴 

#### Jenkinsfile 작성
해당 Jenkinsfile에서는 실제 테스트를 위한 Stage 별로 행위를 작성 하였습니다.
````
#!groovy
 
podTemplate(
  label: "gradle-dev",
  cloud: "kubernetes",
  inheritFrom: "gradle",
  containers: [
  containerTemplate(
      name: "jenkins-jnlp",
      image: "registry.kube-system.svc.cluster.local:5000/jenkins/jenkins-agent-gradle:v13",
      resourceRequestMemory: "1Gi",
      resourceLimitMemory: "2Gi"
    )
  ],
  volumes: [
      hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
  ]
){
node('gradle-dev') {
  stage('Checkout Source') {
    git branch: 'trq1', credentialsId: '1b5ff226-696c-4105-a171-b8f9781020f7', url: 'http://bitbucket.dev.paas-paas.com:7990/scm/test/kube-source-deploy.git'
  }
 
  // Using Maven build the war file
  // Do not run tests in this step
  stage('Build war') {
    sh "gradle clean build"
  }
 
  stage('Build Docker Images') {
    sh "docker build --rm --tag=registry.kube-system.svc.cluster.local:5000/service/wservice:v2 ."
  }
 
  stage('Push Docker Images') {
    sh "docker push registry.kube-system.svc.cluster.local:5000/service/wservice:v2"
  }
 
  stage('Deploy wservice') {
    sh "kubectl set image deployment/wservice wservice=registry.kube-system.svc.cluster.local:5000/service/wservice:v2 -n service"
  }
 }
}
````