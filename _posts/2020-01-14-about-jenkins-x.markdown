---
layout	: posts
title	: Jenkins X Study(1) - Jenkins X 와 GitOps란
summary	: Jenkins
data	: 2020-01-14 22:03:20 +0900
updated	: 2020-01-15 08:50:38 +0900
tag	: jenkins cicd 
toc	: true
comment	: true
public	: true
---
* TOC
{:toc}

# Jenkins X

해당 문서는 작성 중이며 전적으로 개인적인 의견이 많이 반영 되어있다.

----
### 왜 관심을 가졌나?
---
사내에서는 Jenkins를 가지고 CI/CD를 하고 있는 상태이지만 CI/CD 고도화를 하기 위해 다른 CI/CD Tool을 조사하던 중 Jenkins X가 2018년에 릴리즈가 작년에 되었던 것을 기억하게 되어서 자료를 찾고 관심을 가지게 되었다.

관심을 가진 이유는 아래와 같다
- 서비스 환경에 Kubernetes 환경으로 전환 준비 중
- Jenkins에서 Kubernetes 연동시 사용해야할 플러그인 선정(Kunernetes Plugins etc..)
- CI/CD시에 팀 개발자 분들이 좀더 쉽게 이용할 방법이없을까? 

위 내용을 고려하려고 다른 Jenkins를 대체할 툴을 검색 하고 있었지만 기존 환경과 동일하며 Kubernetes에 최적화가 되어있다는 Jenkins X를 보게 되었다.


### Jenkins 와 Jenkins X의 다른점은 무엇인가?
---
가장 큰차이점은 Kubernetes 환경을 위한 추가 도구들을 같이 제공 해준다.
jx Tool을 제공하여 cmd 환경으로 더욱 쉽게 Kubernetes에 배포 할수 있도록 도와준다. 현재 글을 읽을 경우 jx를 사용하면
굳이 Kubernetes를 자세히 이해하지 않아도 배포 할수 있는것으로 보인다. 

[Jenkins X QnA](https://jenkins-x.io/docs/overview/faq/)

위 링크를 보면 Jenkins X 와 Jenkins 비교점을 볼수 있는대, Jenkins의 경우 우리가 흔히 사용한 CI/CD server(당연히 다양한 Plugin을 조합하여 자동화도 가능)를 제공하고 Jenkins X는 Kubernetes 환경에 대한 CI/CD 환경을 통하여 GitOps promotion 과 Pull requests에 대한 Privew 환경을 제공해준다고 한다.

개인적으로 궁금하게 되어서 gitops concept을 이해하기 위해 googling을 하기 시작했다.

### GitOps는 Paradigm 이며, Git 으로 시작해서 Git으로 끝난다.
---
2018년부터 주위에서 GitOps라는 말을 화재가 되어 GitOps 대한 검색을 해보았지만 너무 CI/CD랑 엮어서 고민하게 되다 보니 어렵게 느껴졌다. 하지만 해당 시기에 테스트나 업무로 Kubernetes나 OpenShift 환경에서 Jenkins를 사용하여 CI/CD 자동화를 하는 경우가 많았는대 이때도 배포를 쉽게 하기 위해 모든 시스템 코드를 Gitlab에 넣고 테스트 할때가 많았는데 이게 그 컨셉이였다. 

#### Git을 사용 하여 시스템의 모든 프로세스를 정의 하고 운영한다는게 핵심

각각 환경(dev, stage, production)의 정보가 중앙 repository에 있어 Kubernetes에 쉽게 배포 및 환경 구성이 되게 해야한다는거다. 그래서 GitOps는 Git 과 Kubernetes 붙여서 설명을 많이 하는것을 볼 수가 있었다.

GitOps에서 너무 복잡하게 이해하지 말고 CI는 Master branch에 merging을 업데이트하고 CD는 CI에서 업데이트 된 내용을 Kubernetes에 반영한다고 생각하면 쉬울 것 같다.

1. [What is GitOps](https://www.cloudbees.com/gitops/what-is-gitops)
2. [What is GitOps really](https://www.weave.works/blog/what-is-gitops-really)
