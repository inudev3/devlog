---
emoji:
title: aws - serverless
date: "2023-03-26 18:00:00"
author: inu
tags: aws serverless lambda dynamodb gateway
categories: aws
---

---

author: Inu Jung
pubDatetime: 2023-03-26T15:22:00Z
modDatetime: 2023-03-26T09:12:47.400Z
title: AWS serverless
slug: aws-serverless
featured: true
draft: false
tags:

- aws
- serverles
- lambda
- dynamoDB
- gateway
  description: AWS Serverless 정리

---

서버레스는 `개발자가 관리하지 않아도 되는 것`을 의미하는 범용적인 개념이다. 애초에는 AWS lambda의 `Function as a Service`개념으로부터 출발하였으나, 이제는 AWS가 관리해주는 모든 것을 의미한다.

서버레스 서비스에는

- AWS Lambda
- DynamoDB
- AWS Cognito
- AWS API gateway
- Amazon S3
- AWS SNS&SQS
- AWS Kinesis Data Firehose
- Aurora serverless
- Step Functions
- Fargate
  등이 있다.

## AWS lambda

ec2는 클라우드 상의 가상 서버로, ram과 cpu 제한이 있으며, 종료되기 전에는 계속 실행된다. 스케일링을 위해서 직접 개입하여 서버를 추가/삭제해야 한다.
AWS lambda는 반면 서버가 없는 가상 `function`이다. 짧은 실행시간을 가지며 실행시간에 따라 비용을 지불한다. 사용하지 않을 때에는 실행되지 않는다. 스케일링 또한 별도의 설정없이 자동으로 일어난다.
lambda는 요청 횟수와 시간에 따라서만 비용을 지불하며 프리티어도 관대한 편이다. 또한 AWS의 서비스와 아주 쉽게 통합된다. (프리티어만 관대하지 더 싼건 아니다.) 많은 프로그래밍 언어를 지원하며, CloudWatch로 쉽게 모니터링할 수 있다. function마다 자원을 10GB까지 더 할당할 수 있다. RAM을 늘리면 CPU와 네트워크 처리량도 같이 증가한다.

`lambda container image`는 람다 runtme API를 구현한 이미지로 람다로 실행할 수 있는 컨테이너이다.
유즈케이스의 예시는 S3에 이미지 푸쉬 이벤트가 발생할 경우 람다를 실행하여 해당 이미지의 썸네일을 다시 s3에 푸쉬하고, 이미지 메타데이터를 dynamoDB에 저장하는 것이있다. 혹은 스케줄링할 CRON job 있다면 이벤트브리지로 주기적으로 알람을 발송하고 lambda로 실행할 수 있다. 실행시간 동안만 과금되기 때문에 효율적이다.
지불 방식은 호출당 과금 과 시간당 과금이 있다.

1. pay per call (호출당 과금)
   백만번 호출까지는 무료, 그 뒤부턴 백만번 호출당 0.2달러
2. pay per duration(시간당 과금)
   프리티어에서 400000만 gb-초의 compute time이 무료. gb-seconds는 _1gb X 1초_ 단위로 128mb 펑션은 3200000만 compute time. 그 이후에는 600000 gb-초당 10달러씩 과금

### 람다의 제한점

- 실행환경

1. 메모리 할당은 128mb~10gb 사이이다.
2. 최대 실행 시간은 15분
3. 최대 환경변수 용량은 4kb이다.
4. 펑션에서 사용할 수 있는 디스크 공간("/tmp") 용량은 512mb에서 10gb 까지이다.
5. 1000개의 동시 요청이 가능하다.

- 배포환경

1. 배포할 람다 펑션을 압축했을 때 용량은 최대 50mb, 압축하지 않았을 때는 코드와 의존성 파일들을 전부 합쳐 최대 250mb.
2. /tmp 디렉토리에 실행시 로드할 파일 저장
3. 환경변수 용량은 최대 4kb

람다 펑션은 실행시간의 타임아웃을 설정할 수 있으며 최대는 15분이다. 15분 이상의 펑션은 ec2 등의 환경에서 실행하는 것이 좋다.

### Edge Function

Cloudfront의 edge location에서 실행되는 lambda를 `Edge Function` 이라고 부른다. edge fuction은 cloudfront distribution에서 실행되며, 유저와 근접한 edge location에서 실행함으로써 네트워크 트래픽 지연을 줄이는 것이 사용 목적이다.
`CloudFront` 에서 제공되며, 다음의 2가지 유형이 있다.

1. CloudFront Function
2. Lambda@Edge
   edge function은 cloudfront의 글로벌 네트워크에서 배포되며 서버를 설치/관리할 필요가 없다. 유즈케이스를 예시로 들면 CDN 컨텐트를 커스터마이즈하기 위해 사용된다. 사용되는 만큼만 과금되며 풀-서버리스 이다.
   수많은 유즈케이스가 있다.
3. AB 테스팅
4. 보안, 프라이버스
5. 동적 웹 어플리케이션
6. SEO
7. origin과 data center 라우팅
8. 실시간 이미지 변환
9. 유저 인증/인가
10. 유저 추적/분석

### CloudFront Function

자바스크립트로 쓰여진 `lightweight` 경량 펑션으로, `CloudFront` 가 엣지 로케이션에서 클라이언트 요청(`viewer request`) 을 받아 `origin request` 로 포워딩 하고, 마찬가지로 origin으로 부터 응답을 받아 클라이언트에게 `viewer response`를 포워딩할 때 , *클라이언트 요청*과 _클라이언트 응답_ 두 가지를 수정/변환하는 작업을 한다. `CloudFront` 네이티브 기능으로 `CloudFront`가 전체 펑션/코드를 알아서 관리한다. 초당 100만개의 요청까지 스케일링 되지만 최대 실행시간 1ms 미만의 간단한 펑션만 허용한다.

- 유즈케이스

1. CloudFront Cache Key 정규화
   헤더, 쿠키, 쿼리 스트링, url 등의 요청 어트리뷰트를 변환하여 캐쉬 키 최적화
2. http 헤더 변환
3. url 리디렉트/ 변환
4. 요청 인증/인가
   jwt 등의 유저 토큰 생성 및 검증을 통한 요청 인증/인가

### Lambda@Edge

NodeJS/Python으로 작성되는 펑션이며, 앞서 `CloudFront Function` 클라이언트 요청과 응답만을 변환한다면, `Lambda@Edge`는 오리진에 포워딩하는`origin request` 와 오리진으로 부터 응답받는 `origin response`까지 전부 변환한다. 처리량이 제한적으로 초당 1000개의 요청까지 스케일링되지만, 최대 실행시간 5~10초까지의 복잡한 펑션을 사용할 수 있다. `Lambda@Edge`는 한 AWS Region에서 등록되면 해당 region의 모든 `edge location` 에 복제된다. CPU와 메모리를 조정할 수 있고, aws 서비스나 써드 파티 라이브러리에 의존성을 가지거나 네트워크 액세스를 할 수도 있다. 파일 시스템에 접근하거나 http 본문을 수정할 수도 있다.

### Lambda와 VPC

디폴트로 람다는 유저의 vpc가 아닌 aws 소유의 vpc에서 실행된다. 따라서 자신의 VPC에 있는 서비스에는 접근할 수 없다. 따라서 이를 위해선 람다를 유저의 VPC, 프라이빗 서브넷에 배포하고 보안그룹을 지정하여 ENI(네트워크 인터페이스)를 생성해야 한다. 가장 흔히 쓰이는 유즈케이스는 RDS 프록시이다.

RDS 프록시는 RDS DB 인스턴스의 커넥션을 풀링하는 데에 사용되며, 커넥션 타임아웃을 예방한다. 대략 66%의 장애복구 시간 절감 효과가 있으며 커넥션시 IAM 인증이 필요하도록 설정하여 보안을 강화할 수 있다. RDS 프록시는 프라이빗 서브넷에서만 접근할 수 있기 때문에 람다 펑션이 이에 커넥션을 맺으려면 유저의 VPC에 배포되어야 한다.

RDS(PostgreSQL)와 Aurora MYSQL의 내부 데이터 이벤트로 람다 펑션을 호출하여 데이터를 처리하도록 할 수도 있다. 이 경우 DB에 직접 연결하여 이를 설정해야 하며 aws 콘솔에서는 불가능하다. 또한 DB 인스턴스의 아웃바운드 규칙에 람다 펑션을 허용하고 IAM Policy와 Lambda Resource-based policy로 람다 접근 권한을 부여해야 한다. 람다에 대한 데이터 이벤트는 데이터자체에 대한 이벤트로 DB에 대한 이벤트(데이터와 관련없음)만 전달하는 RDS Event Notification과 구분해야 한다.

## DynamoDB

`cloud-native` 서버레스 DB로 AWS에 의해서 전부 관리되는 NoSQL DB이다. 트랜잭션을 지원한다. 대용량 데이터 및 분산 처리에 적합하다. 매우 빠른 성능을 자랑한다. 인증/인가를 IAM과 통합하여 사용할 수 있다. 저비용 / 자동 스케일링 / 자동 업데이트 를 지원한다. 두 개의 class가 있으며
`Standard` 와 `Infrequent Access(IA)` 이다.

RDS나 Aurora처럼 데이터베이스를 생성할 필요가 없이 바로 사용할 수 있으며 `Table` 단위로 사용한다. 각 테이블은 생성 시에 프라이머리 키를 필요로 하며 테이블 row 개수는 제한이 없다. 각 row(아이템)에는 아무 때나 속성을 부여할 수 있고 row의 최대 크기는 *400kb*이다. 지원하는 데이터 타입은 스칼라 타입, 도큐먼트 타입(리스트,맵 자료구조), 셋 타입을 지원한다. DynamoDB는 스키마가 자주 변경되는 테이블에 적합하다.

읽기/쓰기 처리량에 따라 2가지의 Capacity Mode를 지원한다.

1. Provisioned mode
   테이블의 초당 읽기/쓰기 처리량을 직접 사전에 지정하는 타입이다. 유저가 프로비저닝한 RCU/WCU에 따라 과금되며, auto-scaling을 설정하면 사용량에 따라 자동으로 변경되게끔 할 수 있다.
2. On-Demand Mode
   사용량에 따라 자동으로 프로비저닝 되며 지정할 필요가 없지만 더 비싼편이다. 사용량을 예측할 수 없는 경우에 적합하다. 트랜잭션이 거의 없거나 급작스러운 spot 사용이 있는 경우에도 적합하다.

### DynamoDB Accelerator(DAX)

DynamoDB를 위한 Fully-managed 인메모리 캐시로, 읽기 처리량이 많은 경우 이를 캐싱을 통해 해소할 수 있음. 캐시 데이터에 대한 접근 속도는 매우 빠르며 마이크로세컨드 이내. DAX는 aws가 자동으로 관리해주며 DynamoDB API와 완전히 호환되므로 기존 설정에서 수정할 사항이 없음. DAX 클러스터를 생성해주기만 하면 자동으로 DynamoDb의 데이터를 캐싱하여 동작함. 캐시의 유효기간(TTL)은 디폴트로 5분. `ElasticCache`와의 차이점은 DAX는 개별 오브젝트를 캐싱하므로 쿼리 및 스캐닝에 최적화되며, ElasticCache는 집계 연산 등의 캐싱에 최적화됨

### Stream Processing

item 수준의 변경 사항을 순서가 보장되는 stream으로 변경할 수 있음. 이를 사용하면 DB 변경사항에 대한 처리를 구현하거나 이벤트 처리를 할 수 있음.

- 실시간 변경을 유저에게 이메일 알람
- 실시간 데이터 분석
- 파생 테이블 생성
- 리전 간 복제
- AWS 람다 호출
  등의 유즈케이스가 있음

Stream 처리에는 2가지 옵션이 있는데,

1. DynamoDB stream
   유지기간 24시간, 제한된 consumer. 람다 트리거 처리나 DynamoDB Stream Kinesis Adapter를 사용(뭔지 모름)
2. Kinesis Data Stream
   stream 유지기간 1년, 매우 많은 수의 consumer 가능

### Global Table

복수의 리전에서 DynamoDb를 사용할 수 있게끔 하는 기능으로 Stream 기능을 활성화해야 한다.

### TTL

만료기간 후 삭제하는 옵션으로 세션 핸들링 등에 사용.

### 백업 옵션

1. Point-In-Time 복구(시점 복구)를 사용할 수 있는 지속적 백업
   35일 전까지 백업 됨. 백업 범위 내에서 원하는 시점에 복구 가능(PITR). 복구 시 새 테이블을 생성
2. on-demand 백업
   직접 삭제하지 않는 이상 장기간 보관되는 전체 백업. db 테이블의 성능 및 속도에 영향을 주지 AWS 백업 서비스와 통합하여 사용할 수 있음. AWS 백업 서비스는 라이프사이클 규칙 등을 설정할 수 있고 백업을 관리해주며 백업본을 다른 지역에 복사 할 수 있음. 마찬 가지로 복구 시에는 새 테이블이 생성됨

### S3와 통합

- 내보내기
  DynamoDB 테이블을 S3로 내보낼 수 있음. 이 경우 시점 복구 (PITR) 기능을 활성화해야만 가능하며 35일 전까지의 백업을 내보낼 수 있음.유즈케이스 : Amazon Athena엔진을 사용하여 S3에 쿼리 작업
  s3에 테이블을 내보내도 읽기 성능에는 영향이 없으며, db의 스냅샷을 로깅 목적으로 보관하거나 DynamoDB에서 다시 import 하기 전에 S3에서 데이터 압축/변환 등을 위해 사용. 내보내기 포맷은 DynamoDb JSON / ION이 가능
- 가져오기
  CSV/DynamoDB JSON/ION 포맷 등을 가져올 수 있으며, 쓰기 성능에 전혀 영향을 주지않는다. 가져올 경우 새로운 테이블을 생성하고, 가져오기 중 발생한 에러는 CloudWatch에 보관된다.

## API Gateway

DynamoDB에 대한 CRUD작업을 위한 람다 펑션을 배포했다고 하자. 이 때 클라이언트가 엔드포인트로 접근해서 람다 펑션을 호출하게끔할 수 있는 방법은 여러가지가 있다.

1. 클라이언트에게 직접 IAM Policy를 부여해서 람다 펑션을 호출하게함
2. 람다 펑션과 클라이언트 사이에 ALB를 두고 도메인 엔드포인트 노출
3. 람다 펑션과 클라이언트 사이에 API Gateway를 두고 REST API 노출

API Gateway는 서버리스 서비스로 REST API를 퍼블릭에서 액세스할 수 있게끔 해준다. API Gateway는 클라이언트 요청에 대한 프록시 역할을 하며 인증/인가 및 기타 작업들을 사용할 수 있다. 다음과 같은 특징이 있다.

1. 웹소켓 프로토콜 지원
2. API 버저닝 지원
3. 멀티 환경 지원
4. 보안 기능 수행 가능(인증/인가)
5. 요청 쓰로틀링 처리
6. Swagger/OpenAPI와 통합해 API를 가져오기/내보내기 가능
7. 요청/응답 변환 및 인증
8. API 응답 캐싱
9. SDK/API 명세 생성

API를 aws lambda와 통합하여 lambda를 호출하도록 할 수 있으며 (이 경우 람다의 프록시로 응답은 변환할 수 없음), 람다 외에도 어떤 http 엔드포인트/aws 서비스와도 통합할 수 있다. api gateway 계층을 추가함으로써 캐싱, api 키, 사용자 인증, 쓰로틀링 처리 등의 이점을 누릴 수 있다.

### Endpoint Types

1. edge-optimized
   글로벌 클라이언트를 대상으로 하며, 클라이언트 요청이 디폴트로 `CloudFront edge location` 으로부터 라우팅 된다. API Gateway는 하나의 리전에서만 배포된다.
2. Regional
   클라이언트가 하나의 리전에만 국한된 경우로, API gateway가 배포된 리전에서만 요청을 처리할 수 있다. 이 경우에도 CloudFront Distribution을 생성하여 edge-location 라우팅을 사용할 수도 있다.(캐싱 등 추가작업시)
3. Private
   유저의 VPC 내에서만 접근가능하며 ENI(네트워크 인터페이스)를 할당받아 VPC 엔드포인트를 사용한다. api gateway에한 접근 제어를 위해 `resource policy`를 설정할 수 있다.

### API Gateway Security

gateway에서 유저를 인증할 경우 다음 옵션을 사용할 수 있다.

1. IAM Policy
   aws 서비스에서 접근하는 경우에 유용. 즉 aws 내부에서 사용되는 게이트웨이인 경우
2. Amazon Cognito
   외부 유저를 인증하는 경우
3. 커스텀 인증 기능 구현
   aws 람다를 사용할 수 있음

또한 Amazon Certificate Manager를 사용하여 커스텀 도메인명으로 HTTPS 보안 엔드포인트를 사용할 수 있음. 이경우 필요한 SSL/TLS 인증서는

1. edge-optimized의 경우 us-east-1에 위치
2. regional의 경우 해당 리전에 위치
   또한 Route53에 alias 혹은 cname 레코드로 커스텀 도메인을 가지고 있어야함

API gateway에서 엔드포인트 단위로 어떤 백엔드와 통합할 것인지를 설정하며 (람다 펑션, http 백엔드, aws 서비스, vpc link 등) 람다 펑션의 경우 게이트웨이는 프록시가 되며 리소스로 부터 오는 응답을 변환할 수 없다.  
`edge-optimized` API gateway에서 인증을 처리하는 것과 lambda@Edge / cloudfront function에서 인증을 처리하는 것의 차이점에 유의하자. 전자는 경우 `API gateway` 는 cloudfront `edge location` 이 트래픽을 포워딩하는 `origin` 이 되어 origin에서 인증을 처리하는 것이며, 후자의 경우 `origin`에 전달되기 전, 유저와 가까운 곳에서 별도로 인증을 처리하는 것이다.

api gateway를 배포할 때, deploy stage를 나누어 설정할 수 있다.

### AWS step function

step function은 시각적인 워크플로우를 사용해 aws 람다 펑션 및 aws 서비스를 오케스트레이션 할 수 있는 도구로 aws 람다와 통합된다. 일종의 그래프의 만들어 각 노드가 람다 펑션을 나타내고 노드마다 펑션의 성공/실패 여부에 따라 엣지를 가지치기 할 수 있다.
이 그래프를 사용해 순서/병렬 처리/ 조건부 처리/ 타임 아웃/ 에러 핸들링 등을 설정할 수 있다. 람다 펑션 뿐만 아니라 ec2, ecs, api gateway, sqs 큐 등 매우 많은 aws 서비스를 노드로 표현할 수 있다. 또한 step function 그래프 내에서 `human approval` 단계(step)를 설정하면, 자동화된 워크 플로우 도중 유저가 직접 이를 처리하는 step을 설정할 수 있다.

### Amazon Cognito

외부 유저가 aws 웹/모바일 앱과 상호작용할 수 있는 인증 서비스이다. 두 개의 서비스로 나뉜다.

- Cognito User Pools
  앱 유저에게 로그인 기능을 제공한다. API gateway & ALB와 거의 완벽하게 통합된다.
- Cognito Identity Pools(이전의 Federated Identity)
  유저에게 임시로 AWS 자격증명을 제공하여 aws 리소스에 직접 접근할 수 있는 권한 부여. Cognito User Pools와 완벽하게 통합됨.
  IAM identity는 aws 유저의 권한이며 Cognito가 제공하는 자격증명은 외부 유저에게 부여되는 권한이다. 따라서 불특정 다수, 모바일 유저 등을 위해 사용된다.

1. Cognito User Pools
   aws 웹/모바일 앱을 위한 서버레스 데이터베이스로 단순 로그인 뿐만 아니라 패스워드 초기화, 이메일/전화번호 인증, MFA 인증, Federated Identities를 사용한 로그인도 제공한다(구글, 페이스북 , SAML - OAuth2와 다름)
   API gateway 및 alb와 통합됨. 유저가 우선 Cognito User Pools에 접속하여 토큰을 받으면 해당 토큰을 REST API 접근으로 API gateway에 전달하고 API gateway는 Cognito User Pools으로부터 해당 토큰을 검증함. 검증된 토큰은 백엔드에 유저 자격증명으로써 전달됨.
   API Gateway/ALB와 Cognito User Pools를 함께 사용하면 백엔드는 인증/인가 기능을 완전히 위임할 수 있음
2. Cognito Identity Pools
   Cognito User Pools, 써드 파티 로그인 등을 사용해 AWS 자격 증명을 부여한다. 해당 유저에게 부여할 AWS IAM role 을 Cognito에서 설정해야 하며, 유저의 id에 따라 이를 세분화하여 커스텀할 수도 있다. 특정 유저 id에 iam 자격 증명을 설정하지 않은 경우 디폴트로 설정된 IAM role이 부여된다.
   - 유즈케이스 예시
   1. 외부의 유저가 aws 서비스에 직접 접근하려고 시도하려면, Cognito User Pools에 연결하여 자격 증명 토큰을 받으면, Cognito Identity Pools가 해당 토큰을 검증하고 설정된 IAM role을 부여(없으면 디폴트 IAM role)
   2. DynamoDB의 Security 설정에서 Cognito Identity의 User Id에 따라 특정 row만을 노출하여, 외부 로그인 유저에게 제한적 노출

```toc

```
