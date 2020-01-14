---
layout	: posts
title	: Jenkins X Study(1)
summary	: ${2}
data	: 2020-01-14 22:03:20 +0900
updated	: 2020-01-15 08:16:44 +0900
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
- 조만간 사내 환경에 Kubernetes 환경으로 전환 준비 중
- Jenkins에서 Kubernetes 연동시 사용해야할 플러그인 선정(Kunernetes Plugins etc..)
- CI/CD시에 팀 개발자 분들이 좀더 쉽게 이용할 방법이없을까? 

위 내용을 고려하려고 다른 Jenkins를 대체할 툴을 검색 하고 있었지만 기존 환경과 동일하며 Kubernetes에 최적화가 되어있다는 Jenkins X를 보게 되었다.


***
### Jenkins 와 Jenkins X의 다른점은 무엇인가?
---
가장 큰차이점은 Kubernetes 환경을 위한 추가 도구들을 같이 제공 해준다.
jx Tool을 제공하여 cmd 환경으로 더욱 쉽게 Kubernetes에 배포 할수 있도록 도와준다. 현재 글을 읽을 경우 jx를 사용하면
굳이 Kubernetes를 자세히 이해하지 않아도 배포 할수 있는것으로 보인다.
