---
emoji:
title: aws - s3
date: '2023-03-22 18:00:00'
author: inu
tags: aws s3
categories: aws
---
## S3

S3의 유즈 케이스
- 백업과 스토리지
- 장애 복구
- 아카이브
- 하이브리드 클라우드 스토리지
- 앱 호스팅
- 미디어 호스팅
- 데이터 레이크 & 빅데이터 분석
- 소프트웨어 배포
- 정적 웹사이트

S3에서 유저가 파일을 저장하는 디렉토리를 `버켓` 이라고 하며, 버킷의 이름은 유일해야 한다.(모든 리전과 계정을 합쳐서). 다른 계정이라고 해도 중복될 수 없다.
버킷은 리전 레벨에서 정의된다.

s3에 이름을 정의하고 오브젝트를 업로드하면 public uri가 생성되는데, 처음에는 이 uri로는 오브젝트에 접근할 수 없고, aws credential을 url로 인코딩한 aws pre-signed url을 통해서만 접근가능하다.

S3에 한번에 업로드할 수 있는 파일 크기의 최대는 5gb이며, 그 이상의 크기를 갖는 파일은 multi-part upload를 사용해야 한다.

버킷 내에 폴더를 생성할 수도 있다.

### S3 보안

1. user-based
   IAM-policy: 특정 IAM 유저에게 허용할  api 엔드포인트를 설정하는 보안정책
2. resource-based
   bucket-policies: 버켓 단위로, 특정 유저나 계정에 접근을 허용하는 정책
   아예 오브젝트 단위의 `Object Access Control List`, `Buckets Access Control List` 를 가질 수도 있지만 흔히 사용되지는 않는다.

어떤 IAM 계정이 S3 오브젝트에 접근하려면, IAM-policiy 에서 허용되었거나, buckets-policy에서 허용되었거나, 혹은 IAM-policy에서 명시적으로 접근이 거부되지 않은 경우에는 S3 오브젝트에 접근할 수 있다.

IAM policies를 사용해 특정 IAM 유저, 혹은 IAM role(역할)을 가진 인스턴스에게 접근을 허용할 수도 있다.

bucket-policy를 사용하면 cross-account 액세스, 즉 다른 계정 간 액세스를 허용할 수 있다. 혹은 아예 퍼블릭 액세스를 막을 수도 있다.

s3는 버킷 레벨의 버저닝을 지원하며, 이전 버전으로 롤백/복구 등의 기능을 지원한다. 만약 버저닝을 활성화 하기 전 업로드된 오브젝트가 있다면 버저닝 활성화 이후에는 'null' 로 보전이 표시된다.
버저닝이 활성된 경우 특정 버전의 오브젝트만을 삭제할 수 있다. 버전을 지정하지 않고 오브젝트를 삭제하게 되면, `delete marker`  로 태그된 새로운 버전의 오브젝트를 추가하고 리스트에서 숨긴다. 즉 이전 버전의 오브젝트는 삭제되지 않는다.

버전을 특정해서 삭제해야만 실제로 버킷에서 오브젝트가 삭제된다.

### 정적 웹사이트 호스팅

s3로 정적 웹사이트를 호스팅할 수 있다.

### 지역간, 지역내 복제

지역간 복제를 CRR(Cross-Region Replication),  지역내 복제를 SRR(Same-Region Replication)이라고 하며, 복제는 비동기적으로 이루어진다.

복제는 기본적으로 활성화된 이후 S3에 새롭게 저장된 오브젝트만을 복제하며, 만약 이전 오브젝트들도 복제하려면 `S3 Batch Replication`을 사용하여야 한다.

`delete marker` 가 달린 버전은 복제 시에 같이 복제할 수 있는 옵션이 있다. 그러나 버전 id를 특정한 삭제는 복제 버킷에 복제되지 않는다. 허용할 경우 삭제가 악용될 여지가 있기 때문이다.

또한 복제는 연쇄적으로 일어나지 않는다. 예를 들어 1번 버킷이  2번 버킷으로 복제되고, 2번 버킷이 3번 버킷으로 복제되었다면 1번 버킷의 변경은 2번 버킷까지만 반영된다.

실제로 복제를 수행할 때는 `Replication Rule` 을 생성하여 복제한다.

### 내구성과 가용성

내구성은 천만개의 오브젝트를 만년간 저장 시 1개를 분실할 수 있는 수준을 담보한다. 이는 모든 Storage class에 공통이다.

가용성은 Storage class에 따라 다르다. E3 스탠다드를 예시로 들면 1년에 53분간은 가용하지 않다.

### 스토리지 클래스

1. s3 standard
   99.99% 가용성, 저지연율과 높은 처리량 등으로 자주 접근되는 데이터에 적합. 동시 장애를 2개까지 견딜 수 있음
2. s3  standard infrequent access(S3 standard-IA)
   99.9% 가용성. 저비용이지만 저장 후 조회 시에 비용 발생.  접근 빈도가 낮지만 빠르게 접근해야 하는 데이터에 적합.
3. S3 One-zone IA
   단일 AZ 내에서만 접근되며 높은 가용성(99.999999999%) 을 보이지만 AZ 장애시 소실될 수 있음.
4. Glacier Storage Class
   아카이빙/백업 용도의 저비용 오브젝트 스토리지 클래스
5. Glacier Instant Retrieval
   매우 빠른 조회,검색 속도, 최소 90일간 저장.
6. Glacier Flexible Retrieval
   조회 검색에 걸리는 속도에 따라 expedited (1~5분), standard(3~5 시간), bulk(5~12시간, 무료) 로 나뉨. 마찬가지로 최소 90일간 저장.
7. Glacier Deep Archivee
   장기 스토리지에 적합 . standard 와 bulk가 있으며 최소 180일간 저장
8. Intelligent-Tiering
   이런 많은 storage class 들을 자동으로 프로비져닝 해주는 클래스. 약간의 추가적인 모니터링 비용과 auto-tiering 비용을 내면 사용패턴에 따라 스토리지 클래스를 자동으로 프로비저닝 해줌. 검색/조회 비용이 무료임.


### 라이프사이클 규칙

라이프사이클 규칙에는 다음과 같은 사항들을 정의할 수 있다.
1. 클래스 변경
2. 오브젝트 소멸  - 이전 버전의 파일이나, 로그 파일, 혹은 업로드에 실패한 multipart 파일에 대한 소멸 기간 지정
   라이프사이클의 적용 대상에는 다음과 같은 것들이 있다.
1. 특정 prefix (s3 파일 경로)
2. 특정 오브젝트 태그

S3 Analytics를 사용하면 매일 레포트를 생성하며, standard/standard-ia의 스토리지 클래스에 한해서 더 적합한 클래스를 추천해준다.

### Requester Pays

S3는 버킷 소유자가 오브젝트의 스토리지 비용과 더불어, 버킷의 데이터를 조회하는데 필요한 네트워크 비용까지 지불하는 것이 일반적이다. 그러나 오브젝트의 크기가 크고 트래픽이 높은 경우에 요청자가 이를 지불하게 할 수 있는 옵션이 requester pays이다. 이를 위해서는 요청자 또한 aws에 인증되어야 한다.(aws가 금액을 청구할 수 있는 계정이여야 함)

### Event 알림

s3의 이벤트는 S3로 태그된다. 이벤트가 발생한 오브젝트의 파일명에 따라 이벤트를 필터링할 수도 있다. 알람 전달될 목적지는 SQS(메세징 큐)거나, 람다 펑션, aws sns일 수도 있다.
마지막으로 Amazon EventBridge와 통합될 수 있다. eventbridge에 규칙을 설정하면 eventbridge가 18개 가량 aws 서비스에 알람를 전달하며, json을 활용한 복잡한 필터링이 가능하다.

### 퍼포먼스

S3는 기본적으로 매우 낮은 레이턴시로 요청을 처리할 수 있다.
초당 3500개의 PUT/DELETE/POST/COPY, 5500개의 GET/HEAD 요청을 버킷의 `prefix` 당 처리할 수 있다.
prefix 가 정확하게 어떤 건지 예시로 알아보자.
`bucket` 이라는 버킷을 예로 들면
- `bucket/folder1/sub1/file`  이라는 경로에 file 오브젝트가 있다면 prefix는 bucket/folder1/sub1이다.
- `bucket/folder1/sub2/file`이라는 경로에 file 오브젝트가 있다면 prefix는 bucket/folder1/sub2이다.
  이 모든 prefix 들에 대한 동시요청을 초당 3500개의 PUT/DELETE/POST/COPY, 5500개의 GET/HEAD 를 처리할 수 있다.

- 멀티 파트 업로드
  100mb 이상의 파일에 사용하는 것이 추천되며, 5gb이상이라면 의무적으로 사용해야 한다. 파일을 여러조각으로 나누어 업로드 하며 이 때 병렬화를 통해 속도를 높인다.
- S3 Transfer Acceleration
  S3 버킷과 동일한 리전에 위치한 `Edge Location` 으로 파일을 전송한 뒤 그곳에서 다시 버킷으로 전송함으로써 속도를 향상시키는 방식
- 바이트 범위 Fetch
  GET 요청에 한해 큰 파일의 바이트를 작게 나누어 요청을 병렬화하는 것. 실패 탄력성이 더 높아진다.(실패 시에 더 작은 범위로 다시 시도함)


###  S3 select (Glacier select)

S3에서 오브젝트 전체를 요청하여 많은 네트워크 트래픽을 사용하는 대신 사전에 sql로 요청 오브젝트를 필터링 하여 트래픽을 줄일 수 있다.

### S3 배치 요청
다수의 오브젝트를 대상으로하는 단일 요청은 S3 Batch Operation을 사용할 수 있다. 사용할 operation은 조회/검색, 람다 펑션, 메타데이터/태그 수정, 복제, 암호화 등 다양하며, aws가 재시도나 진행상황 추적, 완료 알림들을 관리해준다.


### S3 오브젝트 암호화

- 서버 사이드 암호화
    1. AWS가 관리하는 키로 암호화 (디폴트)  SSE-S3
       AWS 키를 관리함, 암호화 방식은 AES-256이며, 다음 헤더를 필수로 추가해야함
       "x-amz-server-side-encryption": "AES-256". 새로운 버켓과 오브젝트에 디폴트로 적용됨. 유저가 해당 헤더를 추가해서 오브젝트를 업로드하면, S3가 오브젝트를 S3가 관리하는 키를 사용하여 암호화 후  버킷에 저장
    2.  AWS KMS키에 저장된 KMS 키로 암호화  SSE-KMS
        AWS-KMS 서비스에서 관리하는 키를 사용. 어떤 키를 사용할지 사용자가 선택함. `CloudTrail` 를 사용해 키 사용 기록을 관리할 수 있다는 점도 장점임. "x-amz-server-side-encryption": "aws:kms". 헤더를 필요로함. KMS 키는 별도의 api 호출을 필요로 하며 호출 회수 제한이 있기 때문에(초당 5500,10000건) 쓰로틀링이 발생할 수 있음.
    3. 사용자가 제공한 키로 암호화 SSE-C
       요청 시 사용자가 직접 암호화 키를 전송하고 이를 사용하는 방식. 암호화 키를 헤더에 포함시켜 요청하기 때문에  HTTPS 프로토콜을 사용해야 함. `x-amz-server-side-encryption-customer-algorithm` 헤더를 필요로함
- 클라이언트 사이드 암호화
  Amazon S3 client-side encryption library 등의 암호화 라이브러리를 사용해 사용자가 직접 오브젝트를 암호화 한 뒤 s3에 저장하는 것. 암호화와 복호화가 모두 클라이언트 사이드에서 일어남.

파일을 업로드할 때 업로드 키를 지정하거나, 버킷에서 디폴트 암호화 키를 지정할 수 있다.
클라이언트 사이드 암호화는 콘솔에서 지원하지 않으며 sdk나 cli를 통해서만 사용이 가능하다.

기본적으로 S3에 오브젝트를 업로드하면 자동으로 SSE-S3 방식으로 암호화되지만, 버킷 정책(보안 정책)에서 암호화 헤더를 넣어야만 오브젝트를 업로드할 수 있도록 강제할 수 있다.

### CORS
`cross-origin Resource Sharing` 이 약자. `origin`이란 프로토콜+호스트네임+포트번호를 합친 것을 말한다. (프로토콜이 동일하더라도 보안, 로드밸런싱 등의 목적으로 복수의 포트를 사용하는 경우도 있음)
`http://example.com/app1` 과 `http://example.com/app2` 는 동일한 origin이지만
`http://example.com/app1` 과 `http://otherexample.com` 은 별도의 origini이다.(호스트네임이 다름)
어떤 하나의 origin에서 클라이언트가 요청을 할 때 다른 origin(`cross-origin`)에도 요청이 가는 경우(예: 자바스크립트 fetch 요청),  cross-origin이 해당 요청에 응답하기 위해서는 CORS 헤더(예:`Access-Control-Allow-Origin`) 로 이를 허용해야만 한다.
예를 들어 origin이 응답에 cross-origin의 이미지 리소스를 포함한다고 하자.
웹 브라우저는 먼저 다음 형태의 `pre-flight`  요청을 보낸다.
```http
OPTIONS/
Host:www.otherexample.com
Origin:http://www.example.com
```
이 pre-flight 요청에 대해 `cross-origin`이 CORS를 허용한다면 다음과 같은 pre-flight 응답을 보낸다.
```
Access-Control-Allow-Origin:http://www.example.com
Access-Control-Allow-Methods:GET,PUT,DELETE
```
이렇게 origin의 요청을 cross-origin이 허용한다는 것을 Access-Control-Allow-Origin 헤더로 확인하면 그제서야 정상적으로 요청을 보낸다.
```http
GET/
Host:www.otherexample.com
Origin:http://www.example.com
```

### S3 MFA
S3에서 DELETE 작업을 할 때 MFA 인증을 통해 보안을 강화할 수 있다. 예를 들어 특정 버전을 영구적으로 삭제/버킷 버저닝 비활성화 등. MFA Delete를 하려면 버저닝이 활성화되어 있어야 한다.

MFA DELETE 설정은 콘솔에서는 변경할 수 없으며 IAM credential에 등록된 MFA 기기의 ARN과 code를 직접 커맨드라인에 입력해 sdk와 cli로만 변경할 수 있다

### S3 Access Log

동일 리전 내에 다른 버킷에 대한 모든 요청/접근을 로그를 저장하는 `Logging Bucket`을 만들 수 있다. **주의할 점: 절대 절대 버킷에 자기자신 로그를 저장해서는 안된다!!.**  로그파일 오브젝트가 생성될 때마다 또다른 로깅이 트리거됨 (순환참조) . 타겟 버킷은 logginservice에 대해서 로그 오브젝트를 생성하도록 bucket policy가 업데이트 된다.

### Pre-signed URL

프라이빗 버킷의 가시성을 변경하지 않고 다른 유저가 접근할 수 있게 하기위해 pre-signed url을 사용할 수 있다. pre-signed url을 사용하는 유저는 이를 생성한 유저의 권한을 물려받는다. 생성한 유저가 PUT/DELETE가 가능했다면 사용하는 유저도 pre-signed url을 사용해 PUT/DELETE를 할 수 있다. Pre-signed url은 만료기간을 지정해야 한다.

### Glacier Vault Lock

`S3 Glacier Vault`  는 `S3 Glacier Storage Class` 와 다른 점에 주의하자. Glacier Vault는 스토리지 클래스와는 별도의 아카이빙용 `vault`  컨테이너를 사용하는 스토리지 서비스이다.
`S3 Glacier Vault`  의 특징은 `WORM(Write Once, Read Many)` 전략을 채택했다는 것이다. Vault Lock Policy를 적용하면 이후 이를 변경할 수 없다. 데이터 보존을 위해 사용된다.  Vault Lock Policy가 적용된 이후 저장된 오브젝트는 루트 계정도 이를 삭제할 수 없다.

### S3 ObjectLock

S3 버킷에서도 S3 ObjectLock을 사용해서 비슷한 목적을 달성할 수 있지만, 약간 다르다.  S3 ObjectLock는 오브젝트 수준의 lock이며 버저닝이 활성화된 경우에만 특정 버전을 특정 시간동안   Delete할 수 없게 Lock을 건다.
또한 마찬가지로 루트 유저도 변경할 수 없는 `Compliance` 모드와 루트 유저는 변경가능한 `Governance` 모드가 있다. 두 모드 전부 retention period (만료기간)을 지정해야 한다.
반면 Legal Hold는 retention period / retention mode와 관계없이 무기한으로 오브젝트를 삭제할 수 없게 된다. 그러나 `IAM permission` 의 `S3:PutObjectLegalHold` 를 통해서 Legal hold 자체를 제거하거나 변경할 수 있다.

### S3 Access Points

access points를 사용함으로써 S3 Bucket 보안을 단순화할 수 있음. access 포인트는 각각 고유 도메인명을 가지며 접근할 origin 또한 인터넷/VPC 모두 설정할 수 있음. 단 이 때 VPC origin에서의 접근을 설정하기 위해서는, 해당 VPC에  VPC Endpoing Gateway를 설치해야하며  endoint policy에서  accesspoint와  s3 버킷에 대한 접근을 모두 허용해야함. 즉 endoint policy, accesspoint policy, bucket policy 3개의 보안 정책이 필요.

### S3 Object Lambda

analytics 어플리케이션이 필요한 가공 데이터가 있을 때 이를 aws lambda function을 통해 가공할 수 있다. s3 origin access point로 부터 가져온 데이터를 변경하지 않고, 별도의 `s3 Object Lambda Access Point`  를 설치해 해당 액세스 포인트로 접근하면 . aws lamdba function 실행된다.