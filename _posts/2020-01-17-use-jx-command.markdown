---
layout	: default
title	: Jenkin X Study(2) - jx command 사용해보자 (작성중)
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

---
