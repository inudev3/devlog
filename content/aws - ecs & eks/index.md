---
emoji:
title: aws - ecs& eks
date: '2023-03-23 18:00:00'
author: inu
tags: aws ecs eks
categories: aws
---

### Amazon ECS

도커 컨테이너를 AWS상에서 실행하게 되면 AWS ECS 클러스터 상에서 ECS 태스크를 하나 실행하게 된다. 이 때 클러스터를 구성하는 방식에 따라 LaunchType이 나뉜다.
1. EC2 Launch type
   AWS ECS 클러스터는 EC2 인스턴스들로 이루어진 클러스터로 구성하여, EC2 인스턴스를 유저가 직접 프로비져닝하고 관리할 의무가 있다. (자동 스케일링 X). 각각의 EC2 인스턴스는 `ECS Agent` 를 실행하여 `ECS Service` 와 `ECS Cluster` 에 자신을 등록해야 한다.
   이 과정이 모두 준비가 된다면, ECS 태스크를 실행하고나서부터 해당 도커 컨테이너를 클러스터 내의 인스턴스에서 시작/중지하는 작업을 AWS 서비스가 모두 관리하게 된다.
   이 타입을 사용할 경우 EC2 인스턴스에 대한 프로파일을 정의할 수 있으며, 해당 프로파일은 ECS Agent가 각종 API 호출을 하는 데에 사용된다.(ECS, CloudWatch, ECR, SSM Parameter Store, etc..).
2.  Fargate Launch type
    유저가 인프라스트럭쳐를 프로비져닝하거나 관리하지 않아도 되는 서버리스 런치 타입이다. ECS 태스크를 정의하기만 하면, aws가 필요한 cpu/ram에 따라 클러스터를 구성한다.

- ECS Task Role
  Task를 정의할 때 특정한 작업을 수행하는 Role을 정의할 수 있다. (API 호출 등 )

만약 클러스터에 정의한 복수의 task를 http 엔드포인트로 노출시키고 싶다면, ALB에 앞단에 두어 포워딩하도록 할 수 있다. NLB의 경우 AWS Private Link와 사용하거나 대역폭이 높은 백엔드 task에 경우 권장되며, deprecated된 ELB의 경우 Fargate Launch Type과는 사용할 수 없다.

Amazon ECS에서 영속 데이터를 관리하려면 네트워크 파일시스템인 EFS 파일시스템을 마운트해서 사용한다. EFS는 AZ 단위로 유효하기 때문에 같은 AZ 내의 task는 볼륨을 공유할 수 있다.  Fargate type 에 EFS를 마운트 한다면 서버리스 인프라 구성이 된다. S3는 파일시스템으로 마운트가 불가능하니 주의하자.

ECS 클러스터에서 오토 스케일링 그룹(ASG)을 생성하여 EC2 launch type을 시작할 수도 있다. 해당 ASG는 EC2 콘솔과 ECS 콘솔에서 모두 관리할 수 있다.


클러스터의 EC2 Launch Type에서 ASG를 사용한다면, 오토 스케일링에는 두가지 옵션이 있다.
1. ASG 오토 스케일링
   CPU 사용량으로 오토 스케일링
2. ECS Cluster Capacity Provider
   ECS 태스크를 기준으로 ASG를 스케일링 해줌.
   2번 ECS Cluster Capcity Provider가 조금 더 최신 기능이며 이를 사용하는 것이 좋은 옵션이다.

ECS 서비스에서 task 단위의 오토 스케일링을 사용할 수도 있다.  모든 오토 스케일링 서비스와 마찬가지로
2. target tracking
3. step scaling
4. scheduled scaling
   등의 개념을 지원한다.


### ECS Rolling Update

현재 running 태스크를 유지하면서 새로운 버전의 컨테이너를 업데이트하기 위한 기능
업데이트 중에 실행되어야 할 minimum task 백분율과 rolling update를 위해 추가할 업데이트된 태스크의 maximum task 백분율을 설정하면 minimum task를 유지하면서 순차적으로 task 버전을 교체함. 즉 minimum task가 100% 이상이면 태스크를 중단하지 않고 새로운 버전들을 추가하면서 업데이트를 진행하며 100% 이하이면 기존 태스크를 비율만큼 중단.

### ECS 태스크와 이벤트 브릿지

ECS 태스크를 실행하여 특정 작업을 수행하고 종료할 수 있다면, Amazon EventBridge를 통해 특정 이벤트가 발생할 시 이를 처리하도록 활용할 수 있다.
예를 들어 S3에 upload 이벤트 발생시 EventBridge가 ECS 태스크를 실행하여, 태스크는 버킷에서 오브젝트를 가져와 가공, RDS/Dynamo DB에 저장할 수 있다. 이러한 태스크는 수행할 작업(S3&DynamoDB) 를 task role로써 태스크 생성 시에 정의해야 한다.
혹은 이벤트 브리지를 사용해 주기적으로 scheduled ECS task를 실행할 수도 있다. 배치 프로세싱 등에 적합하다.
혹은 이벤트 브리지 대신 SQS 큐를 사용할 수도 있다. 이 경우 ECS 오토 스케일링을 사용한다면 메세지 수에 따라 태스크가 스케일링 된다.

### Task 정의

Task 정의는 JSON 포맷을 사용하며 다음과 같이 일반적으로 도커 컨테이너를 실행할 때 명시하는 정보를 포함한다.
1. 이미지 이름
2. 포트 바인딩 (도커의 -v 옵션)
3. 필요 메모리 용량과 CPU 코어수
4. 환경변수
5. 네트워크 정보
6. IAM ROLE(권한)
7. 로깅 설정

EC2 Launch Type을 사용하면서 컨테이너 포트만 명시하고 호스트 포트를 명시하지 않는다면 포트가 동적으로 부여되며 계속해서 변경된다. 이 때 ALB를  사용한다면 타겟그룹의 포트를 어떻게 설정할지 고민할 필요 없이 ECS Service가 포트를 매핑해준다. (인스턴스만 명시하면 task 포트는 ECS가 매핑해줌). 이 때 EC2의 보안그룹의 인바운드 규칙은 ALB의 보안그룹에 대해 모든 포트를 허용해야 한다.

Fargate Launch Type를 사용한다면 각각 ECS 태스크가 ENI(네트워크 인터페이스)를 가지며, 각기 고유한 프라이빗 ip가 할당되며, 모든 eni가 태스크와 동일한 포트(80번)에 매핑된다. 따라서 ECS ENI 보안 그룹은 alb에 80번 포트를 인바운드에서 허용해야 한다.

ECS 태스크에서 접근해야하는 AWS 서비스가 있다면, 해당 **태스크를 정의할 때** 접근권한이 있는 `IAM ROLE` 을 부여해야 한다.

환경변수를 설정할 때는 태스크 정의에서 직접 하드코딩한 커맨드나 url을 입력할 수도 있지만, api key나 db 패스워드 등의 민간함 정보는 ssm 파라미터 스토어나 secrets manager 같은 서비스에 저장한 뒤 런타임에 가져오도록 할 수 있다. 이러한 서비스를 이용하면 태스크에 런타임에 저장된 변수를 가져온다. 혹은 Amazon S3 버킷에서 환경변수를 가져오는 방법도 있다.

여러 컨테이너가 스토리지를 공유해야하는 경우가 있다.  예시로 유즈케이스를 들자면  앱 컨테이너가 있고, 해당 컨테이너가 마운트 볼륨에 로그를 쓰면, `audit` 역할을 하는 컨테이너가 이를 읽고 처리하는 경우가 있겠다. 이러한 패턴을 `사이드카 패턴` 이라고 한다. 이 경우 두 컨테이너 모두 해당 경로는 `bind mount` 하여 컨테이너를 실행해야 한다. `EC2 Launch Type`  이라면 바인드 마운트는 단순히 인스턴스의 스토리지(EFS 볼륨/EFS 등)를 그대로 사용하며, 데이터의 생명주기는 인스턴스의 생명주기와 일치한다. `Fargate Launch Type`  이라면 컨테이너 생명주기와 일치한다(일시적)


### 태스크 배치 (EC2 Launch)

EC2 Launch Type에서, ECS 서비스는 `scale-out` 시 새로운 태스크마다 어떤 인스턴스에 배치해야할지를 주어진 인스턴스의 CPU /RAM 용량 및 가용한 포트를 통해 결정한다. 마찬가지로 `scale-in` 할 때에도,  어떤 태스크를 제거할지도 결정해야 한다. 이러한 결정은 ECS가 자동으로 내리지만, 태스크 배치 정책(`task placement strategies`) 혹은 배치 제한 조건(`task placement constraints`)를 설정함으로써 어느정도 유저가 개입할 수도 있다.

ECS는 다음의 순서대로 태스크를 실행할 인스턴스를 고려한다.
1. 태스크 정의에 명시된 CPU / RAM 제한과 포트가 available한 인스턴스를 식별한다.
2.  태스크 배치 한도`task placement constraints`를 만족하는 인스턴스를 식별한다
3. 태스크 배치 정책 `task placement strategies`을 만족하는 인스턴스를 식별한다.

- 태스크 배치 정책 유형
1. BinPack
   가용 CPU/RAM이 가장 적은 인스턴스부터 선택. 즉 먼저 태스크가 배치되어 있는 인스턴스부터 마저 태스크를 한도까지 배치하는 것. 실행 인스턴스의 숫자를 줄여 비용이 절감됨
   다음과 같이 태스크를 pack할 field를 `json` 으로 명시함
    ```json
    "placementStrategy":{
    {
    "field":"memory",
    "type" : "binpack"
    }
    }
    ```
    가용 메모리가 가장 적은 인스턴스부터 마저 배치함. 
2. Random
	말 그대로 랜덤하게 배치
3. Spread
	 명시한 field에 대해서 균일하게 태스크를 배치. 인스턴스의 CPU/MEMORY 뿐만 아니라 AZ 등도 선택 가능

    ```json
     "placementStrategy":{
        {
            "field":"attribute:ecs.availability-zone",
            "type" : "spread"
        }
    }
    ```
    인스턴스의 AZ가 고르게 분포되도록 배치	

- 태스크 배치 제한 조건
1. distinctInstance
   하나의 instance에 태스크가 하나만 배치되도록 함
2. memberOf
   표현식을 사용(`Cluster Query Language`)라고 부름 하여 태스크를 배치할 인스턴스를 지정함
    ```json
    "placementConstraints":{
    {
    "expression":"attribute:ecs.instance-type =~ t2.*",
    "type" : "memberOf"
    }
    }
    ```


### Amazon ECR

Amaon Elastic Container Registry 의 약자로 docker hub와 같은 이미지 레포지토리로써의 기능을 한다. 주로 private으로 이용되지만 public repository도 있으며 ECS와 완전히 통합되고 이미지는 S3에 저장된다.

ECS 태스크가 ECR 레포지토리로부터 이미지를 pull 하려면 태스크에 ECR 레포지토리에 접근할 수 있는 `IAM ROLE` 이 부여되어야 한다. 

ECR은 이미지 버저닝, 태깅, 라이프사이클 관리 및 이미지 취약점 스캐닝을 지원한다.

### AWS Copilot

컨테이너 앱을 빌드/배포 하기 위한 CLI 도구이다. ECS/Fargate/AppRunner에서 앱을 실행하며 필요한 모든 인프라스트럭쳐(ELB,ECS,VPC,ECR..)를 프로비저닝 해준다. CodePipeline을 사용해 자동으로 배포해주며, 여러 환경에 배포할 수 있다.


## AWS EKS

Amazon Elastic Kubernetes Service의 약자. aws상에서 관리되는 쿠베네티스 클러스터를 실행할 수 있는 서비스이다. ECS와 비슷한 목적을 가지지만 완전히 다른 API를 가지고 있으며 어느정도 서로를 대체한다고 할 수 있다.
쿠버네티스는 클라우드에 독립적/비종속적으로, 클라우드간 마이그레이션이 원할하다. 
ECS에서 하나의 Instance 단위는 EKS에서 하나의 node 단위와 의미가 비슷하다. ECS task는 EKS의 pods와 의미가 통한다. 즉, 여러 pod이 하나의 node에서 실행가능하며 여러 node가 모여  worker nodes를 이룬다.(ECS의 클러스터)

- Node Type
1. Managed Node Groups
	EKS가 ASG로써 Node를 생성/관리/스케일링 해줌. Node하나가 하나의 EC2 인스턴스이며 spot-instance나 on-demand를 지원함
2. Self-Managed
	직접 노드를 생성, 등록해야함. ASG에서 관리됨(오토 스케일링) prebuilt AMI를 사용할 수 있음
3. AWS Fargate
	node를 등록하거나 관리할 필요도 없이 그냥 pod(컨테이너)를 실행햐는 서버레스 타입.


- StorageClass
	EKS Cluster 수준에서 storage class 를 명시해야 함. CSI(Container Storage Interface)와 호환되는 드라이버를 사용함
	EBS/EFS(Fargate와 유일하게 사용가능), FSx for Lustre, FSx for NetAPP ONTAP 등이 사용 가능

```toc
```