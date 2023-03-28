---
emoji:
title: aws - messaging
date: '2023-03-25 18:00:00'
author: inu
tags: aws messaging SQS SNS Kinesis
categories: aws
---

## AWS messaging

앱 간의 통신 방법에는 직접 통신하는 synchronous(동기) 방식과  중간 매개체로 큐를 사용하는 asynchronous(비동기) 방식이 있다.

동기 방식의 문제점은 갑작스런 트래픽 증가이다. 트래픽이 갑자기 증가하는 경우 수신하는 앱은 트래픽을 처리하기 버거운 상황이 발생한다.

따라서 어플리케이션 간에 결합을 분리하는 것이 좋다. 이를 위해서는
1. SQS 같은 큐 방식
2. SNS 등의 pub/sub(구독/발행)모델
3. Kinesis 등의 실시간 스트리밍 모델
   을 사용하는 것이 좋다.

## SQS
10년 이상된 aws 서비스임, 어플리케이션 간의 결합도를 낮추기 위해 사용.
큐 내부에 저장할 수 있는 메세지의 수 / 한번 에 전송할 수 있는  메세지 수는 제한이 없지만, 메세지의 유효기간은 디폴트 4일 최대 14일 이다. SQS의 지연율은 10ms보다도 작다. 메세지 하나당 256kb의 크기 제한이 있다. 큐의 제약사항으로 메세지가 두 번 연달아 전달될 수 있고, 순서가 뒤바뀔 수 있다.(순서를 최대한 유지하려고 함)

Producer
SDK의 `SendMesssage API`를 사용해 메시지를 SQS 큐에 전달. 메세지는 수신자가 삭제하기 전까지는 영속함. 한번에 보낼 수 있는 개수에는 제한이 없음

Consumer
큐로부터 메세지를 수신하고(한번에 최대 10개) 이를 처리 후,  SDK의  `DeleteMessage API`를 호출하여 메세지를 큐에서 삭제 .

여러 인스턴스가 동시에 Consumer가 될 수도 있음. 병렬적으로 메세지를 consume함. 이 때문에 `At-least once delivery` 정책 / `Best-effort ordering` 정책을 사용하고 메세지가 중복되거나 순서가 바뀔 가능성이 있음. 수신한 인스턴스는 메세지를 삭제할 의무를 가짐(다른 인스턴스가 똑같은 메세지에 접근하지 않도록

Consumer 인스턴스를 오토 스케일링 그룹 (ASG)으로 설정할 수 있는데, 이 경우 scaling policy는 `Cloudwatch Metric` 에서 큐 길이에 해당하는 `ApproxiateNumberOfMessages`로 정확하게 설정해줘야  한다.

이렇게 SQS 큐를 어플리케이션 계층을 분리하는데 사용할 수 있다. 예를 들어 요청을 받아 비디오를 처리한 뒤 S3 버킷에 저장하는 앱이 ASG에 배포되어 있다고 하면, 처리 작업을 별도의 ASG 앱으로 분리시킨 뒤 기존 엔드포인트의 앱은 요청을 SQS로 전달하고, 새로운 ASG는 이를 consume하도록 하고 큐 길이에 따라 scaling 되게 하면, 두 개의 별도의 작업을 기준으로 독립적으로 스케일링하는 어플리케이션 계층이 만들어진다.

### SQS 암호화
`in-flight` 암호화: HTTPS 사용
`at-rest` 암호화: KMS 키 사용
`client-side` 암호화

### 권한 제어
SQS API에 대한 IAM policy 필요
SQS 권한 정책 - SQS queue에 메세지를 produce할 수 있는 서비스 허용

- Message Visibility Timeout
  메세지가 poll된 뒤 타임아웃(디폴트 30초) 동안 다른 consumer들에게 비공개.
  이 타임아웃 시간 내에 poll된 메세지가 처리되고 삭제되어야함을 의미.  그러나 만약 타임아웃이 지나도 처리되고 삭제되지 않는다면 다시 SQS 큐 내에서 식별되고 다른 consumer가 poll할 수 있음.처리가 오래 걸려 타임아웃을 시간을 늘리거나/ 혹은 처리시간을 짧게 하고 싶은 경우 `sdk`의 `ChangeMessageVisibility API`를 호출해야함. 타임아웃시간이 긴 경우, consumer가 처리에 실패하면(에러) 재시도 시간이 김. 만약 타임아웃시간이 처리시간보다 더 짧은 경우 재시도 횟수가 늘어나 오버헤드가 커짐(consumer가 중복해서 처리할 가능성 상승)

-  SQS Long Polling
   큐가 비어있을 때 메세지를 일정기간동안 기다리는 것을 `Long Polling`이라고 함. 메세지가 없으면 기다리지 않는 것은 `Short Pollinig`이며 Long Polling이 권장됨. 기다리시간은 1초~20초 사이 설정 가능

- FIFO Queue
  선입선출이라는 의미로, 큐의 메세지 순서를 보장하는 것. 앞서 병렬처리로 인해 중복가능, 순서 보장 못함이라는 제약이 있던 일반 SQS 큐와는 달리 중복이 없고 순서가 일치하는 것을 보장함. 하지만 초당 처리량에 제약이 생김(일반큐는 없음) 배치작업하면 3000개/초, 배치 없이 300개/초. 주의점으로 큐 이름이 `.fifo`로 끝나야함

만약 데이터베이스의 초당  트랜잭션  요청 수가 너무 많으면, 트랜잭션이 중간에 분실될 수 있다. 이러한 경우를 방지하기 위해 SQS를 중간 버퍼로 사용할 수 있다. 메세지에 `enqueue`하는 ASG와 `dequeue`하는 ASG를 별도로 두어서 dequeue하는 ASG가 트랜잭션을 RDS/AuroraDB 등의 Data 계층에 insert하는 것이 확인되면 메세지를 큐에서 삭제함.

## Amazon SNS

구독/발행 패턴을 사용하기 위한 서비스.
다수의 수신자에게 직접 메세지를 발신하기 보다,  특정 topic(`SNS topic`)에 메시지를 전달하면 topic이 다시 구독자들에게 *발행*(`publish`)하는 방식

구독자의 수에는 제한이 없고. 이벤트 발행자 (`publisher`) 는 토픽에 메세지를 한번만 보내면 됨/
토픽당 1250만개의 구독자가 가능하다. 토픽은 계정당 10만개의 제한이 있지만 제한을 늘릴 수도 있다.

`SNS`의 구독자(subscriber)는 SQS, aws 람다, `kinesis data firehose` (줄여서 KDF) 등의 aws 서비스가 가능하며 또한 직접 이메일이나 모바일 알림, sms 메시지(문자메세지), https endpoint 호출도 가능하다. `kinesis data stream` 은 구독이 불가능하니 주의하자

SNS의 발행자(event publisher)는 거의 모든 AWS 서비스가 가능하다. 토픽에 이벤트를  발행하려면 SDK를 사용해야 한다.
1. 토픽 생성
2.  구독 생성
3. 토픽에 이벤트 발생

혹은 모바일 앱 sdk를 사용하여 플랫폼 어플리케이션과 엔드포인트를 생성한 뒤, 해당 엔드포인트에 직접 이벤트를 발행하는 방식도 있다.`Direct Publish` 라고 부른다.

서비스에서 사용시 SNS API에 대한 IAM ROLE을 부여해야 하며 SQS나 S3와 마찬가지로 SNS Access Policy를 사용하면 복수의 계정에서 접근하게 할 수 있다.

SNS 토픽에 이벤트를 발생하고 SQS 큐를 해당 토픽에 구독하여, 각각의 서비스마다 하나의 SQS 큐만을 consume하게 할 수 있다. 이렇게 SNS와 SQS를 결합하면 서비스간의 결합을 완전히 끊어낼 수 있다. 이를 `fan-out` 패턴이라고 한다.

이러한 패턴은 매우 유용한데, 예를 들어 S3 이벤트 알람을 발생시킨다고 해보자. 동일한 이벤트 유형(오브젝트 생성) 과 동일한 오브젝트 접두어(/images) 는 동일한 이벤트로 취급된다. 이러한 이벤트를 여러 SQS 큐/ 람다 펑션 등에 전달하고 싶을 때, SNS 토픽에 이를 발행하고 여러 SQS큐를 구독하게끔 하면 된다.

### FIFO Topic

SNS 또한 SQS와 마찬가지로 FIFO를 적용할 수 있는데, 메세지의 그룹 `id` 로 동일 그룹 내의 메세지의 순서를 보장한다. `deduplication`(중복 제거) 기능을 사용할 수도 있다. FIFO를 사용할 경우, topic의 구독자는 `SQS FIFO` 만이 가능하다. 또한 SQS FIFO와 동일하게 초당 처리량에 제한이 있다. SNS FIFO와 SQS FIFO를 결합한 `fan-out` 패턴은 유용하니 알아두자.

### Message Filtering

발행한 메세지 중 구독자에게 전달될 메세지를 미리 필터링할 `JSON policy` 를 설정할 수 있다. 발행된 메세지가 조건을 만족하는 경우에만 구독자에게 전달된다. 따라서 이 필터링 정책은 하나의 구독단위에서 정의된다.

## Kinesis

실시간 데이터의 처리, 분석에 최적화된 서비스. 앱 로그, 지표, IoT 데이터 등 실시간 데이터에 적합하다.

- Kinesis Data Stream - 데이터 스트림의 처리, 저장에 특화
- Kinesis Data Firehose - AWS 데이터 스토리지 서비스에 데이터 스트림 전송
- Kinesis Data Analytics - SQL/Apache Flink를 사용한 데이터 스트림 분석
- Kinesis Video Stream - 비디오 스트림의 처리, 저장

### Shard

데이터 스트림은 여러개의 shard로 분할 되며, shard 개수는 데이터 스트림의 적재(ingestion) 및 소비(consumption) 용량을 나타낸다. 이는 사전에 프로비저닝 되어야 한다.

우선, Kinesis Producer가 데이터 스트림에 `record` 를 생성한다. `record`는 Partition Key와 최대 1mb의 `data blob` (데이터 덩어리)의 키-밸류로 이루어져 있다. 이 중 `Partition Key` 가 해당 레코드가 스트림의 어떤 `shard`에 전달될지를 결정한다. 파티션키는 해싱되어 정수로 변환되고, 각 샤드는 자신의 고유한 정수 범위를 가지고 있으며 파티션 키의 해싱값에 따라 `record` 가 샤드에 분배된다. 프로듀서는 스트림에 샤드 개당 초당 1mb 혹은 초당 1000개의 메세지를 전송할 수 있다. 즉 샤드 개수는 스트림의 데이터 처리량에 따라 프로비저닝 된다.

그 뒤 Consumer가 스트림으로부터 `record`를 전달 받는다. 이 때 `record`에는 `Partition key`와 `data blob` 에 더해 동일 샤드 내의 순서를 나타내는 `sequence No.` 가 추가된다. sequence No.는 샤드에 데이터가 삽입될 시에 부여되는 `monotonnic increasing` 단조증가 값으로, 데이터 스트림의 순서를 보장할 수 있게 해준다.  consumer의 처리량은  하나의 `shard` 당 모든 consumer를 합해 초당 2mb의 처리량을 보장하는 `shared` 모드와 각각의 consumer에게 `shard` 개당 2mb/s 의 처리량을 보장하는 `enhanced`  모드 두가지 유형이 있다.

Kinesis Data Stream의 데이터 유지기간은 하루에서 365일까지 설정가능하며, 유지기간 내에 원하는 데이터를 재처리 할 수 있다.
- *불변성* - 유효기간이 지나지 않는 이상 데이터 스트림에서 데이터를 삭제할 수는 없다.

프로듀서로는 AWS SDK, Kinesis Producer Library(KPL), Kinesis Agent 등이 가능하며 consumer는  직접 AWS SDK, Kinesis Producer Library(KPL)를 사용하거나 aws가 관리하는 Kinesis Data Firehose, Kinesis Data Analytics, AWS lambda 등을 사용할 수 있다.

### Capacity Mode

데이터 스트림의 용량 유형은 두가지가 있다.
1. 프로비저닝할 샤드를 개수를 직접 선택하거나 api를 사용하는 Provisioned Mode에서는 샤드 개당 초당 1mb의 데이터 인풋, 초당 2mb의 데이터 아웃풋을 사용할 수 있으며, 이 때 비용은 시간당 사용한 샤드 개수에 따라 지불한다.
2. On-demand 모드는 직접 프로비저닝 하지 않는 모드로, 사용량에 따라 자동으로 샤드가 스케일링 되며, 샤드당 디폴트로 초당 2mb의 데이터 인풋, 초당 4mb의 데이터 아웃풋을 처리할 수 있다. 지난 30여일간의 사용지표를 토대로 자동으로 스케일링된다. 시간당 스트림 및 데이터 인풋/아웃풋 용량에 따라 비용을 지불한다

### Data Stream Security

데이터 스트림은 리전별로 배포되며 IAM Policy를 통해 스트림에 대한 접근 제어를 할 수 있다. 다른 메세징 서비스와 마찬가지로 https를 통한 `in-flight`, kms를 사용한 `at-rest`, 클라이언트 사이드 암호화가 모두 가능하다. 또한 VPC 엔드포인트를 설정할 수 있어 프라이빗 서브넷이 프라이빗 네트워크로 바로 접근하게끔 할 수 있다. 모든 API 호출은 CloudTrail로 모니터링된다.

### CLI

데이터 스트림을 생성하고 `Capacity Mode` 를 설정했다면, CLI로 데이터를 `produce`하고 `consume` 하는 작업을 할 수 있다. produce 하기 위해서는 다음 커맨드를 입력한다.
```bash
aws kinesis put-record --stream-name test --partition-key user1 --data "user signup" --cli-binary-format raw-in-base64-out
```
consume하기 위해서는 다음 3개의 커맨드를 입력한다.
먼저 stream name으로 스트림의 샤드 정보를 조회한뒤 `shard id`를 조회하여 샤드 `iterator` 의 `hash id`를 조회하고, 해당 `iterator hash id`로 데이터를 consume해야 한다.
```bash
#describe the stream
aws kinesis describe-stream --stream-name test

#get shard iterator
aws kinesis get-shard-iterator --stream-name test --shard-id shardId-000000000000 --shard-iterator-type TRIM_HORIZON

## consume some data
aws kinesis get-records --shard-iterator <샤드 iterator id>
## 매번 consume할 시마다 shard 내의 모든 레코드를 consume하려면 next-shard-itertor 값이 전달되며 이를 다시 한번 사용해야 다음 레코드를 consume할 수 있다
```
위의 방식은 로우레벨한 방식이며, Kinesis Producer Library를 사용하면 직접 iterator id를 조회하지 않고 consume할 수 있다.
위 커맨드는 produce 할 때 데이터를 base64 포맷으로 인코딩했으므로 레코드를 consume하면 아웃풋을 디코딩해야 한다. 한 개의 shard 내의 모든 레코드를 consume하려면  next-shard-iterator로 출력되는 id를 사용해 다시 데이터 consume 커맨드를 입력해야 한다.

### Kinesis Data Firehose

Kinesis Data Firehose는 데이터 스트림 레코드를 입력받아 이를 처리하고 원하는 목적지에 이를 쓰고 저장하기 위한 도구이다. aws lambda function등을 사용해 레코드를 처리하고 목적지에 데이터를 batch write 할 수 있으며 kinesis data stream, amazon cloudwatch, aws IoT 등 다양한 클라이언트로 부터 레코드를 입력받아 처리하고 목적지에 저장한다.
목적지가 될 수 있는 유형은
1. AWS 목적지
   AWS 서비스로 Amazon S3, Amazon RedShift(S3에 쓴 뒤 복사하여 저장), Amazon OpenSearch 등이 있음
2. 써드파티 목적지
   Datadog, MongdoDb, new Relic, Splunk 등의 서드파티 앱에 쓸 수 있음
3. 커스텀 목적지
   유저가 만든 http 엔드포인트 등

Kinesis Data Firehose는 직접 인프라를 등록할 필요가 없는 서버리스 서비스이며 자동으로 스케일링 된다. 사용한 데이터 용량으로 가격을 지불하며 *거의* 실시간이다. 배치(`batch`)로 데이터를 쓸 경우 데이터가 배치 크기에 미달하면 최소 1분의 지연시간, 혹은 데이터 크기가 최소 1mb 이상이여야 쓰기가 시작되기 때문에 `거의`라는 수식어가 붙었다. aws 람다로 데이터 처리가 가능하며 매우 다양한 형태의 데이터 포맷으로 변환/압축 작업을 지원한다. 처리에 실패한 데이터는 S3로 백업할 수 있다.

firehose와 data stream의 차이를 알아두자. data stream은 데이터의 수집/처리를 위한 실시간 서비스로 producer/customer 코드를 직접 작성하며, 샤드를 프로비저닝해야한다. data firehose는 데이터 스트림을 여러 목적지에 전달하기 위한 서비스로, 자동으로 스케일링되며, 배치 쓰기를 위한 대기시간이 존재한다.데이터 재처리가 불가능하다.

data firehose는 aws 상에서 `delivery stream` 메뉴에 있다.

또한 firehose의 목적지마다 버퍼를 사용할 수 있으며, 버퍼의 크기와 버퍼를 `flush`할 주기를 설정할 수 있다. 버퍼는 대용량의 배치 쓰기 등에서 api 호출 횟수를 줄이는 등의 성능 최적화에 도움이 된다.

### Ordering

지금까지의 내용을 바탕으로 messaging 순서에 대해서 논의해보자.
1. Kinesis Data Stream
   Kinesis Data Stream에 produce할 복수의 데이터의 순서를 보장하려면 어떻게 해야할까?
   바로 같은 `partition key` 를 명시해야 한다. 같은 partition key는 동일한 해쉬값 - 동일 shard 를 의미하고, 동일 shard 내에서는 단조증가하는 `sequence No.`로 순서가 보장되기 때문이다. 예를 들어 , 트럭의 주행 데이터를 kinesis data stream에 전달한다고 가정하자. partition key는 트럭마다 고유한 id, 예를 들어 차량번호로 설정하면, 같은 트럭의 주행 데이터는 항상 동일한 shard에 저장됨- 항상 순서가 일치함- 이 보장된다. partition key의 개수 - shard의 개수
2. SQS Queue
   SQS queue에서 순서를 보장하는 방법은 SQS FIFO Queue를 사용하는 것이다. group id를 사용하지 않으면 하나의 Consumer만이 큐를 사용가능하며 , 모든 메세지가 순서대로 정렬된다. group id를 사용한다면  group 개수만큼의 병렬 consumption이 가능하며 group 내에서 순서가 보장된다. Kinesis shard에서 하나의 shard를 병렬적으로 consume할 수 있는 읽기 트랜잭션의 최대 개수는 5개이다.  SQS FIFO 큐의 `group id`와 kinesis data stream의 `partition key`는 비슷하다.


SQS와 SNS 등은 `클라우드 네이티브` 프로토콜로 aws 상에서만 사용할 수 있다. 그렇다면 온프레미스에서 사용하는 amqp, stomp 등의 오픈 메세징 프로토콜을 사용하는 앱이 있다고 할 때, 이를 귿래 aws 상에서 마이그레이션 하려면 어떻게 해야 할까? Amazon MQ를 이용하면 된다. Amazon MQ은 오픈 프로토콜 중 RabbitMQ와 ActiveMQ를 위한 메세지 브로커 서비스 이다. Amazon MQ의 특징 다음과 같다.
1. SQS/SNS 처럼 높은 수준의 스케일링이 불가능함(처리량에 한계가 있음)
2. 서버리스가 아니고 실제 서버가 필요하므로 multi-az 등의 복구 조치 방법이 준비되어야 함
3. SQS와 같은 큐 기능, SNS와 같은 토픽(발행/구독) 기능을 통합한 브로커의 기능을 함
4. 복구 조치를 위해서 동일 리전에 EFS 네트워크 볼륨에 MQ 브로커를 마운트해서 백업함. 장애 시 다른 AZ의 MQ(StandBy)가 해당 EFS 볼륨에 마운트.
 
```toc
```