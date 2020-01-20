---
layout	: default
title	: Jenkin X Study(2) - jx command 설치
summary	: JenkinsX
data	: 2020-01-17 08:09:36 +0900
updated	: 2020-01-17 08:09:36 +0900
tag	: JenkinsX CI/CD jx
toc	: true
comment	: true
public	: true
---
* TOC
{:toc}

# jx Command 란
jx command는 Jenkin X 에서 제공해주는 명령어 이며, 해당 명령어를 사용하여 Kubernetes에 Jenklins X Pipeline(GitOps)를 쉽게 구성해주고 CLI 기반으로 운영 할 수 있게 해준다. 또한 jx 명령어를 사용하여 새로운 Kubernetes Cluster 생성 하여 신규 환경도 구성 가능하다.
---

### jx Command 설정

MacOS 에서 테스트를 할 예정이기 때문에 brew 로 설치하지만 혹시 Linux에서 사용하여 테스트 할 수 있기 때문에 curl 명령어로 설치하는 방법 과 Windows에서 설치하는 방법도 작성 하였다.
```
# brew command 를 사용하여 설치
$brew jenkins-x/jx/jx

# curl 명령어를 사용하여 설치
$curl -L "https://github.com/jenkins-x/jx/releases/download/$(curl --silent "https://github.com/jenkins-x/jx/releases/latest" | sed 's#.*tag/\(.*\)\".*#\1#')/jx-darwin-amd64.tar.gz" | tar xzv "jx"

# jx command를 /usr/local/bin 로 이동
$sudo mv jx /usr/local/bin
```
Windows에 설치 방법 (chocolatey 사용하여 설치 방법)
1. CMD(command Prompt)를 Admin 권한으로 실행
2. powershell.exe를 실행하고 choco binary를 설치 한다 
`@"%SystemRoot%\System32\WindowsPowerShell\v1.0\powershell.exe" -NoProfile -InputFormat None -ExecutionPolicy Bypass -Command "iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))" && SET "PATH=%PATH%;%ALLUSERSPROFILE%\chocolatey\bin"`
3. chcolety를 사용하여 Jenkins X 설치  
`choco install jenkins-x`
	link : [Install jx](https://jenkins-x.io/docs/getting-started/setup/install/)

### jx Commnad를 확인 해보자

Jenkinx X 에서는 쉽게 CI/CD 환경이 구성 가능하다고 하는대 많은 기능중 하나에서는 Kubernetes 환경에서 운영 가능한 Kubernetes Cluster 및 운영에 필요한 툴들을 설치 해준다. Jx 명령어로를 통해서 기존에 있는 Kubernetes Cluster에 배포 하거나 신규 환경 생성을 Kubetnetes를 깊게 이해하지 않아도 쉽게 이용 할 수 있다.

Jx Command를 사용 방법을 습득 하려고 Jenkins X 공식 사이트 및 DevOps 2.6 Toolkit 책을 보면서 실습을 해보고 있다.
1. [jx command](https://jenkins-x.io/docs/getting-started/)
2. [DevOps 2.6 Toolkit ](https://technologyconversations.com/2019/01/28/the-devops-2-6-toolkit-jenkins-x-is-born/)

DevOps 2.6 Toolkit은 작년에 사두고 시간이 나지 않아 이제야 읽어본 책중에 하나이다.

### jx 사용전 Prerequisites
jx 명령어를 사용하기전에 기본적으로 필요한 패키지가 설치 되어있어야 한다. 일반적으로 클러스터 구성및 환경 설정시 helm과 kubectl 명령어를 사용하여 구성하기 때문에 꼭 필요한 패키지 리스트 들이다.

```
1. Git
2. kubectl
3. helm (jx create cluster 명령어 사용ㄱ시 helm v2를 요구한다.)
4. Cloude Provider CLI (ex. aws-cli)
```


### jx create cluster
jx 명령어에서는 Kubernetes 환경에 Cluster를 생성 해주는 옵션을 제공 해주는데 아래 와 같은 Cloud provider에서 제공되는 Kubernetes 환경에 구성이 가능하다.  
아래는 `jx create cluster help` 명령어를 사용하여 확인한 내용을 적어두었다.

Cloud Providers:
>    * aks (Azure Container Service - https://docs.microsoft.com/en-us/azure/aks)
>    * aws (Amazon Web Services via kops - https://github.com/aws-samples/aws-workshop-for-kubernetes/blob/master/readme.adoc)
>    * eks (Amazon Web Services Elastic Container Service for Kubernetes - https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
>    * gke (Google Container Engine - https://cloud.google.com/kubernetes-engine)
>    # icp (IBM Cloud Private) - https://www.ibm.com/cloud/private
>    * iks (IBM Cloud Kubernetes Service - https://console.bluemix.net/docs/containers)
>    * oke (Oracle Cloud Infrastructure Container Engine for Kubernetes - https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengoverview.htm)
>    * kubernetes for custom installations of Kubernetes
>    * minikube (single-node Kubernetes cluster inside a VM on your laptop)
>	* minishift (single-node OpenShift cluster inside a VM on your laptop)
>	* openshift for installing on 3.9.x or later clusters of OpenShift

위의 Cloud Provider 사용시 필요한 Tools를 보여준다.

Dependence tools:
>   * kubectl (CLI to interact with Kubernetes clusters)  
>  * helm (package manager for Kubernetes)  
>  * draft (CLI that makes it easy to build applications that run on Kubernetes)  
>  * minikube (single-node Kubernetes cluster inside a VM on your laptop )  
>  * minishift (single-node OpenShift cluster inside a VM on your laptop)  
>  * virtualisation drivers (to run Minikube in a VM)  
>  * gcloud (Google Cloud CLI)  
>  * oci (Oracle Cloud Infrastructure CLI)  
>  * az (Azure CLI)  
>  * ibmcloud (IBM CLoud CLI) 

실습을 위해 minikube로 구성된 Kubernetes에 신규로 설치 할 거니 아래와 같이 명령어로 실행을 해보았다.

추후 테스트를 위해 AWS eks에 셋팅해서 실제 pipeline 구성 예정이다.

사용 명령어 `jx create cluster minikube`

```
# minikube로 클러스터 생성시 일단 minikube에서 사용할 클러스터 Resource를 지정할수있다.
# 또한 minikube 옵션은 2020년 2월에 사라질 예정이라. jx 로 생성시에 minikube를 start해는걸 권장한다.
# 설치 목적보다도 테스트 목적이기때문에 테스트시 minikube 옵션을 사용하여 클러스터를 생성하였다.
Command "minikube" is deprecated, it will be removed on Feb 1 2020. We now highly recommend you use minikube start instead.
? memory (MB) 4096
? cpu (cores) 3
? disk-size (MB) 150GB
? Select driver: hyperkit
Installing hyperkit
Creating Minikube cluster...
* minikube v1.6.2 on Darwin 10.15.2
...
* Done! kubectl is now configured to use "minikube"
? Select Jenkins installation type: Serverless Jenkins X Pipelines with Tekton
Namespace jx created 
...
Set up a Git username and API token to be able to perform CI/CD
Creating a local Git user for github.com server
? github.com username: <USERNAME>
To be able to create a repository on github.com we need an API Token
Please click this URL and generate a token 

# 해당 URL을 들어가면 셋팅된 권한으로 token을 만들수 있다.
https://github.com/settings/tokens/new?scopes=repo,read:user,read:org,user:email,write:repo_hook,delete_repo

Then COPY the token and enter it below:

? API Token: <TOKEN>
Select the CI/CD pipelines Git server and user
? Do you wish to use github.com as the pipelines Git server: Yes
Creating a pipelines Git user for github.com server
To be able to create a repository on github.com we need an API Token
Please click this URL and generate a token 
https://github.com/settings/tokens/new?scopes=repo,read:user,read:org,user:email,write:repo_hook,delete_repo

Then COPY the token and enter it below:

? API Token: <TOKEN>
Setting the pipelines Git server https://github.com and user name <USERNAME>
Enumerating objects: 1440, done.
Total 1440 (delta 0), reused 0 (delta 0), pack-reused 1440
Setting up prow config into namespace jx
Installing Tekton into namespace jx
Installing Prow into namespace jx
...
```

설치가 완료 된후 jx command를 사용하여 클러스터에 정상적으로 연결되는지 확인 해보자
```
$ jx context
Using namespace 'jx' from context named 'minikube' on server 'https://192.168.64.4:8443'.

$ jx environment
Using environment 'dev' in team 'jx' on cluster 'minikube'.
```
만약 해당 환경은 minikube랑 minikube로 보이지만 혹시 gcloud나 aws의 eks에 셋팅하였다면 해당 환경의 정보를 확인 할수있다.
---
