---
emoji:
title: aws - extra storage
date: '2023-03-24 18:00:00'
author: inu
tags: aws FsX DataSync Snowball
categories: aws
---

AWS의 기타 스토리지 서비스에 대한 문서.

매우 안전한 포터블 디바이스로 aws에서 2가지 유즈케이스가 있음
1. 엣지에서 데이터 처리
   origin에서 가까운 edge location에서 데이터를 수집/처리
   Snowcone/Snowball Edge/Snowmobile
2. AWS로 데이터를 마이그레이션
   Snowcone/Snowball Edge

네트워크 상에서 대량의 데이터를 마이그레이션할 때, 회선의 속도제한이나 대역폭 제한 / 네트워크 사용비용 등의 한계가 존재한다. 이 때 AWS에서 Snow Family에 속한 포터블 디바이스를 직접 클라이언트에게 제공하면 이 장비에 데이터를 load하여 다시 AWS에 장비를 보내면, AWS에 이 데이터를 마이그레이션해주는 것이다. 즉 네트워크 회선의 여러 한계점을 우회하기 위해 사용한다.

### Snowball Edge
데이터 마이그레이션 용으로 S3 오브젝트 스토리지 및 프라이빗 블록 스토리지 제공 , 42tb의 `Compute Optimized` 와 80tb의 `storage optimized` 옵션 존재.

### Snowcone & Snowcone SSD
작고 가벼운 포터블 디바이스, 내구성이 좋고 혹독한 환경에서도 견딤. 8tb hdd인 snowcone과 14tb ssd 인 snowcone ssd가 있음. 엣지 컴퓨팅, 스토리지, 마이그레이션 용으로 사용하며 배터리와 케이블을 필요로 함.

이 모든 디바이스는 EC2인스턴스를 실행하거나 aws 람다 펑션을 실행할 수 있다.


엣지 컴퓨팅이랑 인터넷에 접근할 수 없거나 클라우드 서버로 부터 멀리 떨어진 환경에서 데이터를 처리하는 것을 말한다. 유즈케이스로는 데이터 전처리, 머신러닝, 미디어를 인코딩 등의 작업이 있다.

이전에는 Snow Family 장비들을 CLI에서 관리했지만 최근에는 OpsHub라는 별도의 소프트웨어를 사용하여 AWS 서비스 실행(인스턴스 등), 지표 모니터링, 디바이스 클러스터 구성, 파일 전송 등의 작업을 수행한다.

### Amazon FsX

FxS는 써드파티 파일 시스템을 AWS에서 사용할 수 있게 해주는 aws 서비스로, RDS의 파일 시스템 버전이다. Lustre, NetApp ONTAP, Windows File Server, OpenZFS 등의 파일 시스템을 aws에서 사용할 수 있다


FsX for Windows
SMB 프로토콜과 NTFS/ Active Directory Integration, ACL, user quota 등을 지원한다. 리눅스 EC2 인스턴스에 마운트 될 수 있으며 DFS를 지원하여 여러 파일 시스템을 조인할 수 있다. 온 프레미스 인프라에서 VPN 또는 Direct Connect(aws로 네트워크 회선 설치 )방식으로 접근할 수 있으며 멀티 AZ로 구성할 수 있다. 데이터는 전부 S3에 백업된다. 10gb/s의 속도, 100만 IOPS/ 100PB 데이터까지 지원된다. SSD/HDD 모두 사용 가능한다.

FsX for Lustre
Linux와 Cluster의 합성어로, 대규모 컴퓨팅을 위한 병렬 분산 파일 시스템이다. 머신러닝이나 고성능 컴퓨팅 용도로 사용된다. SSD/HDD 지원, S3와 통합(S3 오브젝트를 파일 시스템으로 읽기 가능, S3 오브젝트로 쓰기 가능). S3 백업.

FsX for NetApp ONTAP
NFS,SMB, ISCSI 프로토콜과 호환. 자동으로 스케일링됨. 매우 빠른 복제속도로 테스트에 적합하며 스냅샷, 복제/압축 기능 등.

FsX for OpenZFS
NFS와 호환


### 파일시스템 배포 옵션
1. Scratch File System
   임시 스토리지, 데이터가 복제되지 않음(영속성 없음, 서버 종료시 삭제).
   일반적인 파일시스템보다 6배 빠른 속도. 빠른 처리 및 비용 최적화(저장 비용 감소)
2. Persistent File System
   이름 그대로 영속성을 가지는 파일 시스템 배포 형태, 동일 AZ 파일 복제 , 분단위의 장애 복구 지원. 장기간 보관에 적합

aws는 온프레미스와 클라우드 인프라를 혼합하여 사용하는 하이브라드 구조를 권장한다.
S3는 aws의 독점적인 스토리지 기술이다. S3의 경우 하이브리드 인프라에서 어떻게 온프레미스가 접근할 수 있을까? 바로 AWS Storage Gateway를 사용할 수 있다.

### AWS Storage Gateway

1. S3 File GateWay
   Glacier를 제외한 S3 버킷을 온프레미스 앱 서버에 연결함. NFS/SMB 등의 네트워크 파일 시스템을 사용하여 온레미스에 마운트하면, S3 File Gateway가 파일 시스템을 `HTTPS` 요청으로 변환하여 S3에 포워딩 / S3에 저장, 혹은 S3 버킷을 HTPS 요청으로 읽어 이를 NFS에 저장. 앱 서버에 입장에서는 파일 시스템을 사용하는 것으로 추상화, S3의 입장에서는 단순 https를 통한 api 호출하는 것으로 추상화.  또한, S3에서 최근에 사용한 데이터를 게이트웨이에 캐싱.
   Glacier를 제외한 다른 storage class들을 지원하며, glacier로의 트랜지션도 lifecycle 규칙 설정으로 가능함. File Gateway가 S3에 접근하기 위해서는 `IAM ROLE`을 부여해야함.  SMB를 사용하는 경우 Active Directory라는 유저 인증 시스템과 aws가 통합되어 AD에 로그인하면 aws에 별도로 로그인하지 않아도 파일 시스템이 s3에 읽고 쓸 수 있음
2. FsX File Gateway
   aws에 배포된 FsX 파일 시스템과 온프레미스 서버를 연결해주는 게이트웨이. 게이트웨이가 없이 VPN/Direct connect로도 접근이 가능하지만, 게이트웨이를 사용하면 FsX에 자주 접근하는 데이터를 게이트웨이에 *캐싱* 해준다는 장점이 있어서 사용됨
3. Volume Gateway
   iSCSI 프로토콜를 사용한 블록 스토리지이며 S3에 백업됨. 온 프레이스 앱 서버의 데이터 볼륨을 EBS 스냅샷으로 백업하여 복구시에 도움이 될 수 있음. Cached Volume은 최근 접근한 일부 데이터를 캐싱하여 저장하며 Stored Volume은 전체 볼륨 데이터셋을 온프레미스에 저장. 즉 iSCIS로 볼륨 게이트웨이를 마운드 하면 게이트웨이가 이를 https 요청으로 EBS로 백업
4. Tape Gateway
   물리적인 테이프 데이터(쓰는 곳이 있음)을 iSCSI VTL(virtual Tape Library)프로토콜을 사용하 Tape Library로  변환하여 Amazon S3와 Glacier에 저장해주는 게이트웨이

만약 온프레미스에 게이트웨이를 설치할 수 없거나, 혹은 게이트웨이를 위한 서버를 추가할 수 없다면 aws에서 하드웨어를 구매할 수도 있다.

### AWS Transfer Family

S3, EFS 등의 시스템에서 파일을 전송할 때 api를 호출하지 않고 FTP 프로토콜만을 이용해서 전송할 수 있다. FTP, FTPS, SFTP 등을 사용가능 하며, 엔드포인트의 시간단위 프로비저닝과 데이터 전송량에 따라 가격이 결정된다. 인프라 스케일링 및 가용성을 전부 고려할 수 있다. 유저가 직접 FTP 엔드포인트에 접근하거나 Route53에 DNS를 등록하여 접근하면, Transfer Family는 `IAM ROLE`을 부여받아 Amazon S3 혹은 EFS에 파일을 전송한다.

### AWS DataSync

온프레미스나 다른 클라우드에서 /로부터 AWS로 데이터를 마이그레이션하기 위해서는 에이전트가 필요하다. AWS 서비스 간의 마이그레이션은 에이전트가 필요없다. Datasync는 마이그레이션을 스케줄링하여 실행하며 파일 시스템의 권한을 그대로 보존한다.

Datasync는 AWS Region에 설치되어야 하며 온프레미스에서 AWS로 마이그레이션 하는 경우  온프레미스 서버에는 에이전트가 설치되어야 한다.  온프레미스의 NFS/SMB 등의 네트워크 파일 시스템과 에이전트가 연결되면, 에이전트와 AWS의 datasync가 TLS 보안 프로토콜을 사용해 통신하고 AWS 서비스에 싱크할 수 있다. 만약 네트워크의 속도/대역폭을 초과한다면 Snowcone 장비를 임대하여 Datasync 에이전트를 실행할 수 있다.

```toc
```