---
layout	: posts
title	: Blue/Green with CodeDeploy on AWS
summary	: devployment
data	: 2020-01-23 19:35:14 +0900
updated	: 2020-01-23 19:35:14 +0900
comment	: true
categories: CI/CD
tags:
  - AWS
  - CodeDeploy
---

* TOC
{:toc}

## Blue/Green with CodeDeploy on AWS

### 정리한 이유
Blue/Green은 무슨 방식이고 왜 써야 하는지는 알지만 정확히 AWS에서 CodeDeploy를 통하여 동작하는 방법을 이해하지 못하여 학습 및 회사 내부 공유 차원으로 작성하게 되었다. 저장용 내용이기 때문에 틀린 부분이 있을 수 있어 계속 업데이트 할 예정이다.

### Blue/Green 배포란
간단히 정리해서 말하면 Blue/Green 배포는 기존 애플리케이션 환경에서 신규로 배포되는 환경으로 트래픽 제어를 통하여 전환 하는 방식을 말한다.

![bluegreen](https://github.com/TRQ1/trq1.github.io/raw/master/assets/images/blue_green_deployments.png )

[BlueGreenDeployment](https://martinfowler.com/bliki/BlueGreenDeployment.html )

즉 하나의 세트(Green)를 더 만들어두고 신규 애플리케이션 버전을 배포한 후 그 환경에서 변경 사항을 반영하여 정상적으로 테스트가 통과 되면 해당 환경으로 전체 트래픽을 전환 하여 서비스를 운영한다. 그러면 기존 세트(Blue)는 무엇을 할까? 그냥 가만히 내버려두는 세트는 아니고 중요한 용도로 사용 된다.  

기존 Green과 동일하게 신규 배포가 발생 되는 경우 배포시 Green 환경에 배포하여 변경 사항 및 테스트를 진행한다. 만약 Green에서 배포된 환경이 이슈가 있으면 Role-back을 할 수 있다.  위 환경이 갖추어 지면 실제적으로 중단 없이 효율적으로 배포가 가능하다.

### AWS에서 Blue/Green 배포

일반적으로 AWS에서 Blue/Green 배포를 하려면 CodeDeploy를 사용 한다. AWS의 CodeDeploy를 사용하여 Blue/Green을 구성하면 EC2, AWS Lambda 와 ECS 에서 지원하고 compute platform마다 동작 방식은 틀리다고 한다. 현재 회사에서 사용 되는 환경은 EC2 환경에 배포 하기 때문에  해당 부분이 어떻게 동작 되는지 알아보자.


#### EC2/On-Premises Compute platform에서 Blue/Green
일반적으로는 EC2 인스턴스에서만 지원을하고 On-Premises 인스턴스들에서는 지원하지 않는다고 한다.

사용하기 위해서는 아래 내역대로 추가 요구 사항을 설정 해야한다.

요구사항:
> - EC2 인스턴스에 IAM(CodeDeploy에 관련된) 설정이 되어 있어야한다.
> - 각 EC2 인스턴스에는 CodeDeploy Agent가 반드시 설치되어 실행 되고 있어야한다.
> - Blue/Green에 사용할 EC2 Auto Scaling group 을 생성하기 위해 ASG 생성을 위한 IAM Role이 설정되어있어야한다.  
 
  
- 아래와 같은 단계로 기존에 배포 그룹에 있는 EC2 인스턴스(Blue)들이 새로운 인스턴스(Green)들로 교체가 된다.
	- Green 환경으로 사용하기 위해 새로운 인스턴스들이 프로비저닝 된다.  
	- 해당 Green 환경에는 최신 애플리케이션 버전이 설치 된다. - 즉 배포하여 새로운 애플리케이션 버전으로 업데이트
	- 설정되어 있는 대기 타임 동안에 지정된 Application 테스트나 시스템 상태를 확인한다.
	- Green 환경에 있는 인스턴스들이 ELB에 등록 되며 트랙픽이 Green 환경으로 전환된다. Blue 환경에 있는 인스턴스들은 ELB에서 등록이 제외 되며 삭제 되거나 대기 시간 만큼 해당 인스턴스들은 대기한다.
 

위 내용은 기본적인  EC2에서의 Blue/Green 스텝이라고 한다.  위에 Overview만 봐서는 1월 23일에 발생 된 이슈를 짐작하기 어려워 좀더 문서를 보도록 하였다.


아래에서 내역을 확인 해보면 2가지 방식을 사용하여 Blue/Green 배포를 배포 그룹에 설정 가능하다.
1. 기존 Auto Scaling Group을 복사 하는 방식: blue/green 배포시에, CodeDeploy는 Green 환경으로 사용 할 인스턴스들을 새로 생성하며 해당 방식을 사용할 경우( 결국 옵션이다.) CodeDeploy는 기존에 명시된 Auto Scaling Group을 Green 환경에서 사용을 한다. – 영문을 보다보니 기계적으로 해석 하게 되는대  배포시에 새로운 Auto Scaling Group을 생성하여 새로 생성된 인스턴스들을 이쪽에 등록하고 배포를 하는 형식을 뜻한다. 해당 방식이 운영 환경에서 사용되는 옵션이다.

2. 수동으로 인스턴스 선택하는 방식: EC2 인스턴스에 태그를 하여 Green 환경에서 사용할 인스턴스를 지정 할수 있다. 기존에 만들어진 ASG나 인스턴스를 지정하여 새로 환경을 생성하지 않고 해당 환경만 사용하게 하는 옵션이다. 




#### CodeDeploy Deployment Hook 프로세스
여기서 그럼 BlueGreen 배포시 CodeDeploy에서 호출되는 hook 방식을 확인해보자.


![bluegreen](https://github.com/TRQ1/trq1.github.io/raw/master/assets/images/codedeployHook.png )


위 와 같은 프로세스로 배포가 된다.

위에 Hook 배포 프로세스 보면 새로 생성된 Green 환경의 인스턴스들은 각 AfterAllowTraffic 상태까지 완료가 되어야 기존의 Blue 환경 인스턴스들이 BeforeBlockTraffic 상태를 시작으로 전환하며 최종적으로 End 형태까지 간다. 

최종적으로 END 프로세스 에서는 기존 Blue 환경이 정상적으로 전환되고 Code Deploy 에서  Auto Scaling 그룹이 배포가 완료된 Green 환경으로 변경이 되고, 지속 시간 설정 값 이후에 기존 Blue Auto Scaling이 삭제 되면서 기존 Instance들도 삭제된다.

위처럼 변경되어야 추후 다시 배포가 일어날시 Green 환경이 Blue가되며 신규로 또다른 Green환경(Auto scaling 그룹 복제)를 하기 때문이다.

#### 장애시 대처 방안

1. CodeDeploy 단계에서 실패 했을 경우
 - DownloadBundle - ValidateService 단계 사이에서 에러가 발생 하는경우
     1. 새로 생성된 Green Autoscaling group 삭제
     2. CodeDeploy 에서 기존(Blue) AutoScaling Group으로 설정되어있는지 확인

 - BeforeBlockTraffic 에서 에러가 발생된 경우
     1. 해당 경우는 특정 인스턴스들에서만 발생되었을 확률이 높음
     2. CodeDeploy에서 AutoScaling Group을 Blue에서 새로 생성한 Green AutoScaling Group으로 변경
     3. 다시 배포 진행이 깔끔함
     4. BeforeBlockTraffic 에서 에러가 발생되는경우 각 Hook의 LifeCycle과 배포 도중에 AutoScaling이 갑작스럽게 일어나서 발생 될수 있는 이슈 이므로 해당 배포시에는 Auto Scaling이 안일어날 시간대에 하는것이 가장 좋은 것으로 판단 됨
