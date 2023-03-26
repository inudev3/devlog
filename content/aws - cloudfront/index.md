---
emoji:
title: aws - cloudFront
date: '2023-03-22 19:00:00'
author: inu
tags: aws cloudfront
categories: aws
---


Content Delivery Network(CDN)으로, edge location에 컨텐트를 캐싱함으로서 유저의 읽기 성능을 향상시키는 것

글로벌에 216개의 에지 로케이션이 있으며 계속 증설 중. 유저가 읽기 요청을 하면 트래픽이 edge location에 먼저 전달되고 edge location의 캐시를 먼저 조회. 캐시가 hit하지 않으면 edge location이 트래픽을 읽기 요청의 목적지(`Origin`이라고 함)에 다시 요청을 하여 응답을 받고 이를 캐싱함.

CloudFront이 `origin` 으로 사용할 수 있는 서비스는 S3 버킷/커스텀 http 등이 있다 . 커스텀 http란 ALB, EC2 인스턴스, S3 정적 웹사이트 등의 AWS 서비스 origin 외에도 어떤 http 백엔드도 가능. Edge Location과 origin은 인터넷이 아닌 프라이빗 네트워크로 통신하기 때문에 매우 빠르고 안전함.
S3 버킷 `Origin Access Control(OAC)` 를 사용해 edge location과의 통신에 대한 보안을 설정하며 edge location을 통한 업로드(ingress)도 가능하다. OAC를 사용하기 위해서는 클라우드프론트 `distribution` arn 번호 등을 명시한 allow bucket policy 를 직접 버킷의 policy에 등록해줘야 한다.

`CloudFront`와 `S3 Region Replica` 등의 지역 간 복제기능의 차이점은 우선 CloudFront는 글로벌 엣지 네트워크를 사용하여 216여개의 edge location과 통신하지만 region replica는 복제를 원하는 지역마다 각각 복제를 설정해줘야 하고, CloudFront는 TTL이 존재하는 캐싱기능으로 실시간 반영이 되지 않지만 region replica는 실시간으로 반영된다는 점이다. 정적 컨텐트는 CloudFront가 유리하며, 저지연율이 필요한 동적인 컨텐트는 `region replica`가 유리하다.

CloudFront는 글로벌 네트워크를 사용하기 때문에 origin은 항상 퍼블릭에 노출되어야 한다.

### Pricing
CloudFront 유저가 edge location에서 전달받는 data 트래픽에 따라 가격을 책정하고 따라 가격을 측정하고, 이 가격은 edge location마다 상이하다. pricing class 중 class 100은 가장 비용이 낮은 리전만을 포함하며 class 200은 가장 비싼 지역을 제외한 플랜이다. class all 은 모든 edge location을 포함한다.


### Geo Restriction
CloudFront Distribution을 생성했다면 (origin, CDN 네트워크 등의 설정) 유저의 geo-location에 따라 접근을 허용(`Allowlist`)하거나 제한(`Blocklist`) 할 수 있다.

### Cache Invalidation
TTL이 만료되기 전까진, origin의 컨텐트가 업데이트 되어도 edge location의 캐시는 업데이트되지 않는다. 이를 무효화하기 위해서 강제로 cache refresh 를 통해서 TTL을  우회할 수 있다.
`CreateInvalidation API` 를 요청함으로써 cache를 초기화하며, 이 때 모든 파일 또는 특정 파일 경로 등 원하는 경로를 지정한다. CloudFront에게 요청된 `CreateInvalidation API` 요청은 모든 edge location에 전달되어 각각의 캐시에서 해당 파일에 대한 cache를 초기화한다.


### Cache Key
edge location은  cache key를 사용해 `origin`에서 전달된 오브젝트를 식별한다.  디폴트로 cache key는 호스트네임과 url의 리소스 부분 (파라미터를 제외한 url)로 이루어져 있다. 이 외의 키를 정의하려면 Cache Policy를 사용할 수 있다. HTTP header/ 쿠키 / 쿼리 스트링 등을 키로 만들 수 있다.
또한 TTL도 설정할 수 있다.Cache-Control / Cache-Expire 헤더등을 사용할 수 있다. 주의할 점은 Cache Key로 설정한 모든 값들은 origin에 대한 요청에 그대로 포함된다는 점이다.
쿼리스트링에 대한 policy 옵션은
- all
- whitelist - 특정 쿼리 스트링만을 포함
- include all except - 블랙리스트, 특정 쿼리 스트링만을 제외
- none - 포함안함


### Origin Request Policy
Origin Requests Policy는 Cache Key에서는 제외하지만 origin requests에는 포함시킬 header, 쿠키, 쿼리 스트링 등을 지정하는 옵션이다. 혹은 유저 리퀘스트에서 포함되어 있지않은 커스텀 헤더를 추가할 수도 있다.

### Cache Behaviour
cache behaviour란 edge locations에 컨텐츠가 어떻게 캐싱되고 제공될지를 결정하는 규칙이다. url 패턴이나 요청 메서드 등으로 캐시가 어떻게 동작할지를 결정한다. 즉 어떠한 url 패턴/ 요청 메서드와 origin requests가 매핑될지를 결정하는 규칙이다. 캐쉬 정책과 캐쉬 동작 규칙을 구분하도록 하자. 캐쉬 정책은 일반적인  정책으로 여러 `cloudfront distribution` / `cache behavior` 에 적용될 수 있으며 `cache behavior`는 하나의 cloudFront distribution에서 정의된 규칙이다. 캐쉬 정책이 캐쉬를 어떻게 저장할지를 결정한다면 캐쉬 동작은 캐쉬 요청이 어떻게 origin 과 매핑될지를 결정한다. 캐쉬 동작 규칙은 가장 상세한 규칙(url 패턴 등)에서부터 가장 일반적인 규칙 순서대로 매칭된다.

### CloudFront Signed URL

CloudFront Distribution에 대해서 보안인증을 적용해야할 경우가 있다. 이 경우 인증받은 유저에게  signed url을 제공할 수가 있다. signed url은 origin의 단일 리소스에 대한 인증이며, signed cookie는 origin의 여러 리소스에 대해 인증을 제공한다.  
CloudFront Signed URL을 이용하면 CloudFront의 모든 캐싱 정책이 적용되며, 계정 소유의 url로서 루트 계정에게 관리권한이 있다.
S3 pre-signed url과 비교하면, S3 pre-signed url은 해당 url을 사인한 계정의 권한을 그대로 물려받아 직접 S3에 요청한다는 차이점이 있다. 반면 CloudFront Signed URL 은 해당 CDN(edge location)에 요청한다.

### URL Signing Process

사인할 수 있는 계정은 두가지 유형이 있다.
1. trusted key group
   추천되는 방식으로, API를 사용하여 키를 생성/분배하며, IAM을 통해 관리할 수 있다.
2. 계정이 CloudFront Key Pair 소유
   잘 사용하지 않는 방식으로, root 계정이여야만  CloudFront Key Pair를 소유할 수 있기 때문에 추천되지 않는다.

trusted key group은 `CloudFront distribution` 단위로 구분되며, `CloudFront distribution`은  공개키, 어플리케이션은 비밀키를 나누어 가진다.

### Origin Group
origin 그룹은 origin의 `failover` 장애 복구를 위해 하나의 primary origin과 secondary origin을 묶은 그릅을 참조하는 것이다.  primary와 secondary를 다른 리전 /AZ 에 설치함으로써 지역 장애를 복구할 수 있으며, s3의 경우 primary의 replica를 secondary로 사용할 수 있다.

### AWS Global Accelerator

AWS Global Accelerator 여러 서버가 하나의 동일한 ip를 가지고, 트래픽을 가장 근거리의 서버에 포워딩하는 `Anycast IP` 를 사용하는 방식이다 . 여러 서버가 제각기 다른 고유한 ip를 가지는 방식을 `Unicast IP`라고 한다. 여러 edge location이  Anycast IP를 할당받고, 가장 가까운 edge location에 트래픽이 포워딩 된다. 장애 복구를 위해 `AWS Global Accelerator`는 계정당 2개의 정적 Anycast IP를 갖는다. 고정된 ip를 할당하기 때문에 cache key도 변하지 않는다. 글로벌 aws 네트워크를 사용하며 연결된 app에 대한 헬스체크 또한 제공한다.

`AWS Global Accelerator`와 CloudFront의 차이점을 정리하자면 `AWS Global Accelerator`는 일종의 프록시로서 어플리케이션에 대한 트래픽을 포워딩하거나, 헬스체클를 통해 가용성을 보장하는 것이라면 CloudFront는 캐싱의 목적을 갖고 사용된다. 또한 TCP/UDP 등의 non-http 요청에 `AWS Global Accelerator`가 보다 적합하며 , 빠른 장애조치(2개의 Anycast IP)를 제공한다.

### Field level encryption

HTTPS와 비대칭키를 이용해 한 겹의 보안계층을 추가하는 것. POST 요청의 필드 레벨에서 민감한 정보를 암호화할 때, 유저와 가장 가까운 edge location에서 공개키를 사용해 암호화를 수행한다. 최종 목적지인 웹서버에서 비공개키를 사용하여 이를 복호화하고 데이터를 처리한다.

```toc
```