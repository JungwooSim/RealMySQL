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

#### 03. 아키텍처(<a href="https://github.com/JungwooSim/RealMySQL/architecture" target="_blank">링크</a>)
#### 04. 트랜잭션과 잠금(<a href="https://github.com/JungwooSim/RealMySQL/transactionsandlock" target="_blank">링크</a>)
