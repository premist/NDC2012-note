# Lessons learn from managing database system on Cloud for Social Game
##### 이준섭(Nexon America Senior Database Administrator)

#### 소개
- 메이플스토리, 컴뱃 암즈, 던전앤 파이터 DB 구축
- 삼성SDS DBA
- 데이터베이스 롤백

#### Nexon America
- 3~4개의 소셜 게임 운영중
- 어떻게 하면 좋은 성능을 내는 소셜게임용 DB를 만들 수 있을까?
- 기본적으로 퍼블리셔, 스튜디오나 개발팀을 지원해주는 역할을 함

#### 목차
- Prologue
- Membase Server @ Amazon Cloud
	- General Knowledge
	- Sizing, Monitoring etc
- MySQL for social game
- Conclusion

#### Prologue
###### Client Game Database Life Cycle and Characteristic
- MySQL이나 MSSQL, Oracle을 쓰는 건 선택
- Internal IDC - 리소스 사용이 자유롭다 
	- Memory, CPU, 저장 장치의 제한이 없다 - 고사양 서버를 구축 가능 
- 초기에 만들어진 Initial System을 계속 이용하는 경우도 있고, 사용자가 늘게 되면 서버의 제원을 높이거나 새로운 월드(채널)을 만든다
- 작은 서버의 데이터베이스를 마이그레이션 해야하는 문제도 있음 - 결국 기존 서버를 이용하게 됨

#### 소셜 게임의 다른 점
- 개발 기간이 매우 짧다
- Cloud 환경에서는 Mem/Disk/CPU의 제한이 있기 마련 - Scale Out(수평적인 서버의 확장)/Scale In

#### 기준
- 모두 Cloud 환경에서 구축
- 저비용
- Scale In/Out의 자유로움, Downtime 최소화
- 높은 성능

#### 데이터베이스 후보
- Cloud / Cost, Performance / Scale InOut 기준으로 분류
- MySQL : Internal IDC와의 환경과 비슷하기 때문에 Scale In/Out이 불가능하고, 고성능 인스턴스를 써야 하기 때문에 비용이 많이 든다. Sharding 솔루션이 있으면 데이터를 분산할 수는 있지만, 짧은 시간 때문에 불가능
- MySQL RDS : MySQL의 제한과 동일하고, 시스템 상에서 컨트롤 할 수 있는 부분이 적다. 하지만 Amazon의 API를 이용하여 인스턴스를 자유롭게 조절할 수 있다. 다만 인스턴스의 크기 제한이 있고, RAID 등의 설정이 불가능하기 때문에 MySQL보다 좋지 않은 점도 있음
- MySQL NDB 클러스터 : Cloud에서의 성능이 보장되지 않다. MySQL보다 나은 성능을 보이긴 하지만, Scale In/Out이 MySQL보다는 낫다. 또한 2TB의 사이즈 리밋이 있다.
- Membase(CouchDB) : Cloud에서의 성능이 보장되고, Scale In/Out이 쉽게 가능하고, 저사양의 저비용 서버로도 트렌드를 빨리 따라가는 시스템을 만들 수 있다.

#### Membase Server on Amazon Cloud
- Simple : 클릭 5번으로 서버 빌드. 서버 사양도 몰라도 된다. 자신의 사이즈에 맞는 서버를 사용하면 된다. 하지만 시스템에 딱 맞는 시스템을 구하기 힘들다.
- Fast : 물리 서버를 구축하려면 최소 1~3달이 걸리지만, 서버 주문을 위해 시간을 낭비할 필요가 없다.
- Elastic : 손쉽게 Scale In/Out이 가능하다. 
- 서버를 만드는 과정을 스크립트로 만들어 놓으면 3분 안에 인스턴스를 만들 수 있고, 병렬 처리가 가능하기 때문에 100대의 인스턴스도 3분 안에 deploy가 가능하다. 
- 빠른 시간 안에 서버 안정화가 가능하다.

#### Sizing Membase
- 주요 질문 : 서버(노드)의 수는?
- 네트워크 대역폭
- Disk I/O
- 데이터를 반드시 압축해서 넣어야 한다! 
- **RAM의 사이즈** : 모든 데이터가 메모리상에 올라가게 되고, 0.00x초의 Response Time을 가지므로 매우 빠르다
- 노드 수 : 전체 필요한 데이터의 용량(Cluster RAM Quota required) / 각 서버별 가용 램 용량(Per node RAM Quota)
- Cluster RAM Quota required : (total metadata + working set)*(1 + headroom percentage) / (high water mark percentage)

#### Membase Sharding
- Cluster(논리적인 서버의 그룹) 내에 서버 빌드, 버킷(유저 데이터베이스를 저장하는 전체 공간)생성
- 버킷 내에 해시된 값들이 vBucket으로 저장됨
- Cluster 안에 서버를 더 추가 : Pending -> Active -> 데이터 분산
- 데이터 분산이 되면 동일한 갯수의 데이터가 각각의 서버에 존재하게 됨
- Cluster 내의 모든 서버가 죽어버리면? Restore를 해 주어야 한다

#### Membase Replica (복제)
- Cluster 내에 서버가 하나만 있다면 Replica가 생성되지 않는다
- 서버가 두개라면, 서버1에는 서버2의 복제본이, 서버2에는 서버1의 복제본이 저장된다 (Replica 1) - Server 하나가 내려가도 다른 서버가 역할을 해줌
- Replica 2 : 각 서버에 다른 서버 2개의 백업 데이터를 저장
- Replica : 서버가 내려갔을 때 데이터를 보존할 확률
- 데이터 용량과 업데이트 할 때 들어가는 비용은 풀어야 할 과제

#### Membase High/Low Water Mark
- 시스템 메모리의 80%를 Membase가, 20%를 시스템이 사용
- Low Water Mark : 60% - RAM에서 데이터가 추출되기 시작한다
- High Water Mark : 75% - 마스터 데이터가 RAM에서 추출되기 시작한다
- Low/High Water Mark를 넘었다고 해서 서버를 반드시 추가해야하는 것은 아니다 - Disk에서 읽는 것도 이점

#### Membase Warm-up Process
- 서비스를 재시작하면 10~15분 정도가 소요
- 서버를 재시작하면 전체 데이터를 올리는 데 3시간 정도가 소요 - 다 올라가기 전까지는 서버가 운영되지 않음
- 새로운 서버 추가 -> 기존 서버 옮김 -> 리밸런싱
- 서버가 많을수록 Warm-up process가 짧아지므로 많은 서버를 운영하는 것이 좋음

#### Monitoring Membase Server
- 콘솔로 많은 인스턴스를 관리 가능
- High Water Mark / Low Water Mark의 관리

#### MySQL Server for Social Game
- 특정 게임 플레이 로그를 위한 DB
- 설정 파일
- 결제 로그 데이터베이스

#### Game database server
- Internal IDC
- MySQL 5.0 버전 이상 (5.0 이상이 되면 엔터프라이즈 버전만 멀티스레딩을 지원)
- 메모리와 CPU
- Disk는 RAID10
- Slave 서버는 두 개로 둘 것 : 백업 서버와 통계 서버, Slave에도 RAID10

