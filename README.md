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

// img-1

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

// img-2

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

// img-3

MySQL에서 사용되는 메모리 공간은 글로벌 메모리 영역과 로컬 메모리 영역으로 구분할 수 있다.

글로벌 메모리 영역의 모든 메모리 공간은 MySQL 서버가 시작되면서 무조건 운영체제로부터 할당된다. MySQL의 파라미터로 설정해 둔 만큼 운영체제로부터 메모리를 할당받는다고 생각하는 것이 좋다.

글로벌 메모리 영역과 로컬 메모리 영역의 차이는 MySQL 서버 내에 존재하는 많은 스레드가 공유해서 사용하는 공간인지 아닌지 구분되며 각각 다음과 같은 틍성이 ㅇ

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
