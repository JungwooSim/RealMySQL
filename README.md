# RealMySQL

## 2.3 서버 설정

MySQL 서버는 단 하나의 설정 파일만 사용한다.

리눅스를 포함한 유닉스 계열에서는 my.cnf 를 사용, 윈도우 계열에서는 my.ini 를 사용한다. (이 파일의 이름은 변경할 수 없다)

MySQL 서버는 시작될 때만 이 설정 파일을 참조하는데, MySQL 서버는 여러 개의 디렉터리를 순차적으로 탐색하면서 처음 발견된 my.cnf 파일을 사용하게 된다.

### 2.3.2 MySQL 시스템 변수의 특징

MySQL 서버는 기동하면서 설정 파일의 내용을 읽어 메모리나 작동 방식을 초기화하고, 접속된 사용자를 제거하기 위해 이러한 값들을 별도로 저장해 둔다.

시스템 변수(설정) 값은 글로벌 변수와 세션 변수로 나누어 진다. 각 시스템 변수에 포함된 5가지 선택사항의 의미는 아래와 같다.

- Cmd-Line
    - MySQL 서버의 **명령행 인자로 설정될 수 있는지 여부**를 나타낸다.
- Option file
    - MySQL 의 설정 파일인 **my.cnf(또는 my.ini)로 제어할 수 있는지 여부**를 나타낸다.
- System Var
    - 시스템 변수인지 아닌지 여부
- Var Scope
    - 시스템 변수의 적용 범위
- Dynamic
    - 시스템 변수가 동적인지 정적인지 구분하는 변수

---

## 03. 아키텍처

### 3.1 MySQL 아키텍처

> 이 장의 목적 : MySQL의 쿼리를 작성하고 튜닝할 때 필요한 기본적인 MySQL 의 구조를 훓는데에 있다.

<img src="/img/img-1.png" width="500px;" />

MySQL 서버는 크게 **MySQL 엔진**과 **스토리지 엔진**으로 구분할 수 있다. 이 둘을 합쳐 MySQL 또는 MySQL 서버라고 표현한다.

**MySQL 엔진**

클라이언트로부터의 **접속 및 쿼리 요청을 처리하는 커넥션 핸들러**와 **SQL 파서** 및 **전처리기**, **쿼리의 최적화된 실행을 위한 옵티마이저**가 중심을 이룬다.

성능 향상을 위해 MyISAM의 키 캐시나 InnoDB의 버퍼 풀과 같은 보조 저장소 기능이 포함돼 있다.

**스토리지 엔진**

요청된 SQL 문장을 분석하거나 최적화하는 등 DBMS의 두뇌에 해당하는 처리를 수행하고, 실제 데이터를 디스크 스토리지에 저장하거나 디스크 스토리지로부터 데이터를 읽어오는 부분을 전담한다.

**핸들러 API**

쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때는 각 스토리 엔진에게 쓰기 또는 읽기를 요청하는데, 이러한 요청을 핸들러(Handler)요청이라 하고 여기서 사용되는 API를 핸들러 API라고 한다.

InnoDB 스토리지 엔진 또한 이 핸들러 API를 이용해 MySQL 엔진과 데이터를 주고 받는다.

### 3.1.2 MySQL 스레딩 구조

<img src="/img/img-2.png" width="500px;" />

MySQL 서버는 **프로세스 기반이 아니라 스레드 기반으로 작동**하며, 크게 포그라운드(Foregroud) 스레드와 백그라운드(Background) 스레드로 구분할 수 있다.

**포그라운드 스레드(클라이언트 스레드)**

최소한 MySQL 서버에 접속된 클라이언트 수 만큼 존재.

각 클라이언트 사용자가 요청하는 쿼리 문장을 처리하는 것이 임무다.

클라이언트가 사용자가 작업을 마치고 커넥션을 종료하면, 해당 커넥션을 담당하던 스레드는 다시 캐시(Thread pool)로 돌아가게된다.

이 때, 스레드 캐시에 일정 개수 이상의 대기 중인 스레드가 있으면 스레드 캐시에 넣지 않고 스레드를 종료시켜 일정 개수의 스레드만 스레드 캐시에 존재하게 한다.

이렇게 스레드 개수를 일정하게 유지하게 만들어주는 파라미터가 thread_cache_size 이다.

**백그라운드 스레드**

InnoDB는 여러 가지 작업이 백그라운드로 처리된다. 대표적으로 인서트 버퍼(Insert Buffer)를 병합하는 스레드, 로그를 디스크로 기록하는 스레드, InnoDB 버퍼 풀의 데이터를 기록하는 스레드, 데이터를 버퍼로 읽어들이는 스레드, 그리고 기타 여러 가지 잠금이나 데드락을 모니터링하는 스레드가 있다. 이러한 스레드를 총괄하는 메인 스레드도 있다.

### 3.1.3 메모리 할당 및 사용 구조

<img src="/img/img-3.png" width="500px;" />

MySQL에서 사용되는 메모리 공간은 글로벌 메모리 영역과 로컬 메모리 영역으로 구분할 수 있다.

글로벌 메모리 영역의 모든 메모리 공간은 MySQL 서버가 시작되면서 무조건 운영체제로부터 할당된다. MySQL의 파라미터로 설정해 둔 만큼 운영체제로부터 메모리를 할당받는다고 생각하는 것이 좋다.

글로벌 메모리 영역과 로컬 메모리 영역의 차이는 MySQL 서버 내에 존재하는 많은 스레드가 공유해서 사용하는 공간인지 아닌지 구분되며 각각 다음과 같은 특성이 있다.

**글로벌 메모리 영역**

클라이언트 스레드의 수와 무관하게 일반적으로 하나의 메모리 공간만 할당하게 된다.

생성된 글로벌 영역이 N개라 하더라도 모든 스레드에 의해 공유된다.

**로컬 메모리 영역**

세션 메모리 영역이라 표현하기도 한다.

클라이언트 스레드가 쿼리를 처리하는 데 사용하는 메모리 영역이다.

클라이언트가 MySQL 서버에 접속하면 MySQL 서버에서는 클라이언트 커넥션으로부터의 요청을 처리하기 위해 스레드를 하나씩 할당하게 되는데, 클라이언트 스레드가 사용하는 메모리 공간이라고 해서 클라이언트 메모리 영역이라고도 한다.

클라이언트와 MySQL 서버와의 커넥션을 세션이라고 하기 때문에 로컬 메모리 영역을 세션 메모리 영역이라고도 표현한다.

로컬 메모리는 각 클라이언트 스레드별로 독립적으로 할당되며 절대 공유되어 사용되지 않는다는 특징이 있다.

각 쿼리의 용도별로 필요할 때만 공간이 할당되고 필요하지 않은 경우에는 MySQL이 메모리 공간을 할당조차도 하지 않을 수 있다.

로컬 메모리 공간은 커넥션이 열려 있는 동안 계속 할당된 상태로 남아 있는 공간도 있고(커넥션 버퍼, 결과 버퍼) 쿼리를 실행하는 순간에만 할당했다가 다시 해제하는 공간(소트 버퍼나 조인 버퍼)도 있다.

### 3.1.4 플러그인 스토리 엔진 모델

<img src="/img/img-4.png" width="500px;" />

### 3.1.5 쿼리 실행 구조

<img src="/img/img-5.png" width="500px;" />

**파서**

파서는 사용자 요청으로 들어온 쿼리 문장을 토큰(MySQL 이 인식할 수 있는 최소 단위의 어휘나 기호)으로 분리해 트리 형태의 구조로 만들어 내는 작업을 의미

**전처리기**

파서 과정에서 만들어진 파서 트리를 기반으로 쿼리 문장에 구조적인 문제점이 있는지 확인한다.

각 토큰을 테이블 이름이나 컬럼 이름 또는 내장 함수와 같은 개체를 매핑해 해당 객체의 존재 여부와 객체의 접근권한 등을 확인하는 과정을 이 단계에서 수행한다.

**옵티마이저**

옵티마이저란 사용자의 요청으로 들어온 쿼리 문장을 저렴한 비용으로 가장 빠르게 처리할지 결정하는 역할을 담당하는데, DBMS의 두뇌에 해당한다고 볼 수 있다.

**실행 엔진**

옵티마이저가 두뇌라면 실행 엔진과 핸들러는 손과 발에 비유할 수 있다.

실행 엔진은 만들어진 계획대로 각 헨들러에게 요청해서 받은 결과를 또 다른 핸들러 요청의 입력으로 연결하는 역할을 수행한다.

**핸들러(스토리엔진)**

핸들러는 MySQL 서버의 가장 밑단에서 MySQL 실행 엔진의 요청에 따라 데이터를 디스크로 저장하고 디스크로 읽어 오는 역할을 담당한다.

핸들러는 결국 스토리 엔진을 의미하며 MyISAM 테이블을 조작하는 경우에는 핸들러가 MyISAM 스토리지 엔진이 되고, InnoDB 테이블을 조작하는 경우에는 핸들러가 InnoDB 스토리지 엔진이 된다.

### 3.1.6 복제(Replication)

<img src="/img/img-6.png" width="500px;" />

데이터베이스의 데이터가 갈수록 대용량화돼 가는 것을 생각하면 확장성(Scalability)은 DBMS에서 중요한 요소인데, MySQL 은 확장성을 위해 여러가지 기술을 제공하지만 그 중에서 가장 일반적인 것이 복제(Replication) 이다.

복제는 2대 이상의 MySQL 서버가 동일한 데이터를 담도록 실시간 동기화 하는 기술이다.

일반적으로 MySQL의 복제에는 INSERT나 UPDATE와 같은 쿼리를 이용해 데이터를 변경할 수 있는 MySQL 서버와 SELECT 쿼리로 데이터를 읽기만 할 수 있는 MySQL 서버로 나뉜다.

MySQL에서는 쓰기와 읽기의 역할로 구분해서 쓰기는 마스터(Master), 읽기는 슬레이브(Slave)라 한다.

일반적으로 MySQL 서버의 복제에서는 마스터는 반드시 1개이며 슬레이브느 1개 이상으로 구성될 수 있다.(마스터나 슬레이브라는 것은 단지 그 서버의 역할 모델을 지칭할 뿐이다.)

**마스터(Master)**

기술적으로는 MySQL의 바이너리 로그가 활성화되면 어떤 MySQL 서버든 마스터가 될 수 있다.

마스터 서버에서 실행되는 DML(데이터를 조작하는 문장)과 DDL(스키마를 변경하는 문장) 가운데 데이터의 구조나 내용을 변경하는 모든 쿼리 문장은 바이너리 로그에 기록한다.

그 후 슬레이브 서버에서 변경 내역을 요청하면 마스터 서버에서 바이너리 로그를 읽어 슬레이브로 넘기게 된다.

**슬레이브(Slave)**

슬레이브 서버의 I/O 스레드는 마스터 서버에 접속해 변경 내역을 요청하고, 받아 온 변경 내역을 릴레이 로그에 기록한다.

그리고 슬레이브 서버의 SQL 스레드가 릴레이 로그에 기록된 변경 내역을 재실행(Replay)함으로써 슬레이브의 데이터를 마스터와 동일한 상태로 유지한다.

**복제를 할때 주의해야 하는 점이 있다.**

1. 슬레이브는 하나의 마스터만 설정 가능 : MySQL의 복제에서 하나의 슬레이브는 하나의 마스터만 가질 수 있다.
2. 마스터와 슬레이브의 데이터 동기화를 위해 슬레이브는 읽기 전용으로 설정
3. 슬레이브 서버용 장비는 마스터와 동일한 사양이 적합
  - 마스터 서버에서 수많은 동시 사용자가 실행한 데이터 변경 쿼리 문장이 슬레이브 서버에서는 하나의 스레드로 모두 처리돼야 한다.
    그래서 변경이 매우 잦은 MySQL 서버일수록 마스터의 사양보다 슬레이브 서버의 사양이 더 좋아야 마스터에 동시에 여러 개의 스레드로 실행된 쿼리가 슬레이브에서 지연되지 않고 하나의 스레드로 처리될 수 있다.
    또한, 슬레이브 서버는 마스터 서버가 다운된 경우 그에 대한 복구 대안으로 사용될 때도 많기 때문에 사양을 동일하게 맞추는 경우가 대부분이다.
4. 복제가 불필요한 경우에는 바이너리 로그 중지
5. 바이너리 로그와 트랜잭션 격리 수준(Isolation level)
  - 바이너리 로그 파일은 어떤 내용이 기록되느냐에 따라 STATEMENT 포맷 방식과 ROW 포맷 방식이 있다.
    STATEMENT 방식은 바이너리 로그 파일에 마스터에서 실행되는 쿼리 문장을 기록하는 방식이며, ROW 포맷은 마스터에서 실행된 쿼리에 의해 변경된 레코드 값을 기록하는 방식이다.

### 3.1.7 쿼리 캐시 (Query Cache)

<img src="/img/img-7.png" width="500px;" />

쿼리 캐시(Query Cache)는 타 DBMS에는 없는 MySQL의 독특한 기능 중 하나로서 적절히 설정만 해두면 상당한 성능 향상 효과를 얻을 수 있다.

쿼리의 결과를 메모리에 캐시해 두는 기능인데, 간단한 키와 값의 쌍으로 관리되는 맵(Map)과 같은 데이터 구조로 구현돼 있다.

키를 구성하는 요소 가운데 가장 중요한 것은 쿼리 문장 자체이고 값은 해당 쿼리의 실행 결과이다.

**쿼리 캐시의 결과를 내려주기 전에 반드시 다음과 같은 확인 절차를 거쳐야 한다.**

1. 요청된 쿼리 문장이 쿼리 캐시에 존재하는가?
  - 비교 방식은 요청된 쿼리 문장 자체가 동일한지 여부를 비교하는 것이다.
    여기서 비교하는 대상으로는 공백이나 탭과 같은 문자까지 모두 포함되며, 대소문자까지 완전히 동일해야 쿼리로 인식한다.
2. 해당 사용자가 그 결과를 볼 수 있는 권한을 가지고 있는가?
3. 트랜잭션 내에서 실행된 쿼리인 경우, 그 결과가 가시 범위 내의 트랜잭션에서 만들어진 결과인가? (InnoDB의 경우)
  - InnoDB의 모든 트랜잭션은 각 트랜잭션 ID를 갖게 된다. 트랜잭션 ID는 트랜잭션이 시작된 시점을 기준으로 순차적으로 증가하는 6바이트 숫자값이어서 트랜잭션 ID 값을 비교해보면 어느 쪽이 먼저 시작된 트랜잭션인지 구분할 수 있다.
    InnoDB에서는 트랜잭션 격리 수준을 준수하기 위해 각 트랜잭션은 자신의 ID보다 ID 값이 큰 트랜잭션에서 변경한 작업 내역이나 쿼리 결과를 참조할 수 없다. 이를 트랜잭션 가시 범위라고 한다.
4. 쿼리에 사용된 기능(내장 함수나 저장 함수 등)이 캐시돼도 동일한 결과를 보장할 수 있는가?

   4.1 CURRENT_DATE, SYSDATE, RAND 등과 같이 호출 시점에 따라 결과가 달라지는 요소가 있는가?

  - SYSDATE(), RAND() 같은 함수는 동일 사용자가 동일 쿼리를 실행하더라도 호출하는 시간에 따라 결과가 달라 진다.
    호출될 때마다 결과값이 달라지는 내장 함수, 스토어드 함수 등은 사용하지 않는 편이 쿼리 캐시의 효율을 높이는데 도움된다.

   4.2 프리페어 스테이트먼트의 경우 변수가 결과에 영향을 미치지 않는가?

  - 쿼리 문장 자체에("?")가 사용되기 때문에 쿼리 문장 자체로 쿼리 캐시를 찾을 수가 없다.
5. 캐시가 만들어지고 난 이후 해당 데이터가 다른 사용자에 의해 변경되지 않았는가?
6. 쿼리에 의해 만들어진 결과가 캐시하기에 너무 크지 않은가?
  - 쿼리 캐시의 전체 크기를 64MB로 설정했는데, 만약 어떤 쿼리 하나가 60MB 정도의 쿼리 결과를 만들어내면 하나의 쿼리 때문에 쿼리 캐시를 다 소모해 버릴 수 있다.
    이러한 현상을 예방하고자 특정한 크기 미만의 쿼리 결과만 캐시하도록 설정하는 시스템 파라미터가 있다. (query_cache_limit)
7. 그 밖에 쿼리 캐시를 사용하지 못하게 만드는 요소가 사용됐는가? (아래는 쿼리 캐시를 사용하지 못하게 하는 요소이다.)
  - 임시 테이블(Tempoary Table)에 대한 쿼리
  - 사용자 변수의 사용 : 쿼리에 사용자 변수를 사용하면 프리페어 스테이트먼트와 동일한 효과가 발생하므로 MySQL이 쿼리 캐시를 사용하지 못하게 된다.
  - 칼럼 기반의 권한 설정
  - LOOK IN SHARE MODE 힌트 : SELECT 문장의 끝에 붙여서 조회하는 레코드에 공유 잠금(읽기 락)을 설정하는 쿼리
  - FOR UPDATE 힌트 : SELECT 문장의 끝에 붙여서 조회하는 레코드에 배타적 잠금(쓰기 락)을 설정하는 쿼리
  - UDF(User Defined Function)의 사용
  - 독립적인 SELECT 문장이 아닌 일부분의 서브 쿼리
  - 스토어드 루틴(Procedure, Function, Trigger)에서 사용된 쿼리(독립적인 쿼리라 하더라도)
  - SQL_NO_CACHE 힌트 : SELECT 문장에 SELECT 키워드 뒤에 붙이는 힌트로서, 이 힌트가 사용되면 쿼리 캐시를 사용하지 않는다.

### 3.2 InnoDB 스토리지 엔진 아키텍처

InnoDB는 MySQL에서 사용할 수 있는 스토리엔진 중에서 거의 유일하기 레코드 기반의 잠금을 제공하고 있다. 때문에 높은 동시성 처리가 가능하고 안정적이고 뛰어나다.

<img src="/img/img-8.png" width="500px;" />

### 3.2.1 InnoDB 스토리지 엔진의 특성

**프라이머리 키에 의한 클러스터링**

모든 테이블은 기본적으로 프라이머리 키를 기준으로 클러스터링되어 저장된다.

즉, 프라이머리 키 값의 순서대로 디스크에 저장된다는 뜻이며, 이로 인해 프라이머리 키에 의한 레인지 스캔은 상당히 빨리 처리될 수 있다.

**잠금이 필요 없는 일관된 읽기(Non-locking consistent read)**

MVCC(Multi Version Concurrency Control)라는 기술을 이용해 락을 걸지 않고 읽기 작업을 수행

**외래 키 지원**

InnoDB 스토리지 엔진 레벨에서 지원하는 기능으로 MyISAM이나 MEMORY 테이블에서는 사용할 수 없다.

외래 키는 여러 가지 제약사항 탓에 인해 실무에서 잘 사용하지 않기 때문에 필수적이지는 않지만 개발 환경의 데이터베이스에서는 좋은 가이드 역할을 해줄 수 있다.

실무에서 잘 사용하지 않는 이유는 부모테이블과 자식 테이블 모두 해당 컬럼에 인덱스 생성이 필요하고, 변경 시에는 반드시 부모 테이블이나 자식 테이블에 데이터가 있는지 체크하는 작업이 필요하므로 잠금이 여러 테이블로 전파되고, 그로 인해 데드락이 발생할 때가 많다.

**자동 데드락 감지**

그래프 기반의 데드락 체크 방식을 사용하기 때문에 데드락이 발생함과 동시에 바로 감지되고, 감지된 데드락은 관련 트랜잭션 중에서 ROLLBACK이 가장 용이한 트랜잭션을 자동적으로 강제 종료해 버린다.

그러므로 데드락 때문에 쿼리가 제한시간(Timeout)에 도달한다거나 슬로우 쿼리로 기록되는 경우는 많지 않다.

**자동화된 장애 복구**

손실이나 장애로부터 데이터를 보호하기 위한 여러 가지 메커니즘이 탑재되 있다.

그러한 메커니즘을 이용해 MySQL 서버가 시작될 때, 완료되지 못한 트랜잭션이나 디스크에 일부만 기록된 데이터 페이지(Partial write)등에 대한 일련의 복구 작업이 자동으로 진행된다.

**오라클의 아키텍처 적용**

오라클 DBMS의 기능과 상당히 비슷한 부분이 많다. 대표적으로 MVCC 기능이 제공되는 것과 언두(Undo) 데이터가 시스템 테이블 스페이스에 관리된다는 것이다. 그외에도 여러가지 있다.

### 3.2.2 InnoDB 버퍼 풀

가장 핵심적인 부분으로 디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해 두는 공간이다.

쓰기 작업을 지연시켜 일괄 작업으로 처리할 수 있게 해주는 버퍼 역할도 같이 한다. (INSERT나 UPDATE 그리고 DELETE와 같이 변경된 데이터를 모아서 처리하게 되면 랜덤한 디스크 작업의 횟수를 줄일 수 있다.)

### 3.2.3 언두(Undo) 로그

UPDATE 문장이나 DELETE와 같은 문장으로 데이터를 변경했을 때 변경되기 전의 데이터(이전 데이터)를 보관하는 곳이다.

크게 두 가지 용도로 사용되는데, 첫 번째 용도가 바로 위에서 언급한 트랜잭션의 롤백 대용이다. 두 번째 용도는 트랜잭션의 격리 수준을 유지하면서 높은 동시성을 제공하는 데 사용된다.

### 3.2.4 인서트 버퍼(Insert Buffer)

RDBMS에서 레코드가 INSERT되거나 UPDATE될 때는 데이터 파일을 변경하는 작업뿐 아니라 해당 테이블에 포함된 인덱스를 업데이트하는 작업도 필요하다.

그래서 InnoDB는 변경해야 할 인덱스 페이지가 버퍼 풀에 있으면 바로 업데이트를 수행하지만, 그렇지 않고 디스크로부터 읽어와서 업데이트해야 한다면 이를 즉시 실행하지 않고 임시 공간에 저장해 두고 바로 사용자에게 결과를 반환하는 형태로 성능을 향상시키게 되는데, 이때 사용하는 임시 메모리 공간 인서트 버퍼(Insert Buffer)라고 한다.

### 3.2.5 리두(Redo) 로그 및 로그 버퍼

변경된 내용을 순차적으로 디스크에 기록하는 로그 파일이다. 일반적으로 RDBMS에서 로그라 하면 이 리두 로그를 지칭하는 경우가 많다.

MySQL 서버 자체가 사용하는 로그 파일은 사람들의 눈으로 확인할 수 있는 내용이 아니라서 편집기로 열어볼 수 없으며, 열어볼 필요도 없다.

사용량(특히 변경 작업)이 매우 많은 DBMS 서버의 경우에는 이 리두 로그의 기록 작업이 큰 문제가 되는데, 이러한 부분을 보완하기 위해 최대한 ACID 속성을 보장하는 수준에서 버퍼링하게 된다.

이러한 리두 로그 버퍼링에 사용되는 공간이 로그 버퍼이다.

**ACID는 데이터베이스에서 트랜잭션의 무결성을 보장하기 위해 반드시 필요한 4가지 요소(기능)를 의미한다.**

- Atomic : 원자성 작업이어야 함을 의미
- Consistent : 일관성을 의미
- Isolated : 격리성을 의미
- Durable : 한번 저장된 데이터는 지속적으로 유지돼야 함을 의미
