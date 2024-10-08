# 데이터 타입

칼럼의 데이터 타입과 길이를 선정할 때 가장 주의할 사항은 다음과 같다.

- 저장되는 값의 성격에 맞는 최적의 타입 선정
- 가변 길이 칼럼은 최적의 길이 선정
- 조인 조건으로 사용되는 칼럼은 똑같은 데이터 타입으로 선정

## 문자열(CHAR와 VARCHAR)

### 저장 공간

CHAR와 VARCHAR의 공통점은 문자열을 저장할 수 있는 데이터 타입이라는 점이고, 가장 큰 차이는 고정 길이냐 가변 길이냐다.
  
하나의 글자가 CHAR 타입에 저장될 때는 추가 공간이 더 필요하지 않지만 VARCHAR 타입에 저장할 때는 문자열의 길이를 관리하기 위한
1~2 바이트의 공간을 추가로 더 사용한다. VARCHAR 타입의 길이가 255바이트 이하이면 1바이트만 사용하고, 256바이트 이상으로 설정되면
2바이트를 사용한다. VARCHAR 타입의 길이 관리 최대 바이트는 2바이트로, 즉 VARCHAR 타입의 최대 길이는 65,536 바이트이다.
  
사람들은 보통 길이가 고정적일 때 CHAR를 선택하고, 길이가 가변적일 때 VARCHAR를 선택한다. 하지만 실제로 길이가 정적이냐
가변적이냐만으로 CHAR와 VARCHAR 타입을 결정하는 것은 적절하지 않다.

- 저장되는 문자열의 길이가 대개 비슷한지
- 칼럼의 값이 자주 변경되는지

CHAR와 VARCHAR 타입의 선택 기준은 값의 길이도 중요하지만, 해당 칼럼의 값이 얼마나 자주 변경되느냐가 기준이 돼야 한다.

```sql
CREATE TABLE tb_test (
    fd1 INT NOT NULL,
    fd2 CHAR(10) NOT NULL,
    fd3 DATETIME NOT NULL
);

INSERT INTO tb_test (fd1, fd2, fd3) VALUES (1, 'ABCD', '2011-06-27 11:02:11');
```

![image](https://github.com/user-attachments/assets/9d15f078-2ebd-4a36-bc70-22367d874fa4)
![image](https://github.com/user-attachments/assets/2608f439-0b60-41a5-bbb9-5f20ba2e4780)

여기서 만약 fd2 칼럼의 값을 "ABCDE"로 UPDATE했다고 가정한다면

- CHAR(10) 타입에서는 fd2 칼럼을 위해 공간이 10바이트가 준비돼 있으므로 그냥 변경되는 칼럼의 값을 업데이트만 하면 된다.
- VARCHAR(10) 타입에서는 fd2 칼럼에 4바이트 밖에 저장할 수 없는 구조로 만들어져 있다. 그래서 "ABCDE"와 같이 길이가 더 큰 값으로 변경될 때는 레코드 자체를 다른 공간으로 옮겨서(Row migration) 저장 해야 한다.

물론 주민등록번호처럼 항상 값의 길이가 고정적일 때는 당연히 CHAR 타입을 사용해야 한다. 또한 값이 2~3바이트씩 차이가 나더라도 자주 변경될 수 있는
부서 번호나 게시물의 상태 값 등은 CHAR 타입을 사용하는 것이 좋다. 자주 변경돼도 레코드가 물리적으로 다른 위치로 이동하거나 분리되지 않아도 되기 때문이다.

### 저장 공간과 스키마 변경(Online DDL)

VARCHAR 데이터 타입을 사용하는 칼럼의 길이를 늘리는 작업은 길이에 따라 매우 빠르게 처리될 수도 있지만 어떤 경우에는 테이블에 대해 읽기 잠금을 걸고
레코드를 복사하는 작업이 필요할 수도 있다.

```sql
CREATE TABLE test (
    id INT PRIMARY KEY,
    value VARCHAR(60)
) DEFAULT CHARSET=utf8mb4;
```

```sql
ALTER TABLE test MODIFY value VARCHAR(63), ALGORITHM=INPLACE, LOCK=NONE;
Query OK, 0 rows affected (0.00 sec)

ALTER TABLE test MODIFY value VARCHAR(64), ALGORITHM=INPLCAE, LOCK=NONE;
ERROR 1846 (0A000): ALGORITHM=INPLACE is not supported. Reason: Cannot change column type INPLACE. Try ALGORITHM=COPY.
```

value 칼럼의 길이를 VARCHAR(63)으로 늘리는 경우는 잠금 없이 매우 빠르게 변경되지만, VARCHAR(64)로 늘리는 경우는 INPLACE 알고리즘으로
스키마 변경이 허용되지 않는다. 
  
VARCHAR(64)로 변경하는 경우에는 다음과 같이 COPY 알고리즘으로 스키마 변경을 실행했으며, 스키마 변경 시간도 상당히 많이 걸리게 된다. 
COPY 알고리즘의 스키마 변경은 읽기 잠금까지 필요로 한다. 즉 스키마 변경을 하는 동안 test 테이블에는 INSERT나 UPDATE, DELETE를 실행할 수 없게된다.
  
이러한 차이가 발생하는 이유는 VARCHAR 타입의 컬럼이 가지는 길이 저장 공간 크기 때문이다. utf8mb4 문자 집합을 사용하는 VARCHAR(60) 칼럼은
최대 길이가 240바이트이기 떄문에 문자열 값 길이를 저장하는 공간이 1바이트면 되지만, VARCHAR(64) 타입은 저장할 수 있는 
문자열의 크기가 최대 256 바이트이기 때문에 문자열 길이를 저장하는 공간의 크기를 2바이트로 변경해야 한다.
  
문자열 길이를 저장하는 공간의 크기가 바뀌게 되면 MySQL 서버는 스키마 변경을 하는 동안 읽기 잠금을 걸어서 아무도 데이터를 변경하지 못하도록
막고 테이블의 레코드를 복사하는 방식으로 처리한다. 레코드 건수가 많은 테이블에서 읽기 잠금을 필요로 하는 스키마 변경을 실행하기 위해 스키마를 변경할 때마다 
서비스의 가용성이 훼손될 수 있다.

## 콜레이션

콜레이션은 문자열 칼럼의 값에 대한 비교나 정렬 순서를 위한 규칙을 의미한다. MySQL의 모든 문자열 타입 칼럼은 독립적인 문자 집합과 콜레이션을 가진다.
각 칼럼에 대해 독립적으로 지정하지 않으면 MySQL 서버나 DB 기본 문자 집합과 콜레이션이 자동으로 설정된다.

### 콜레이션 이해

문자 집합은 2개 이상의 콜레이션을 가지고 있는데, 하나의 문자 집합에 속한 콜레이션은 다른 문자 집합과 공유해서 사용할 수 없다.
또한 테이블이나 칼럼에 문자 집합만 지정하면 해당 문자 집합의 기본 콜레이션이 해당 칼럼의 콜레이션으로 지정된다.

- `utf8mb4_0900_ai_ci`
  - 3개의 파트로 구성된 콜레이션 이름
  - 첫 번째 파트: 문자 집합의 이름
  - 두 번째 파트: 해당 문자 집합의 하위 분류
  - 세 번째 파트: 대문자나 소문자의 구분 여부를 나타낸다.
    - `ci`이면 대소문자를 구분하지 않는 콜레이션이며, `cs`이면 대소문자를 별도의 문자로 구분하는 콜레이션이다.
  - `ai` 또는 `as`는 액센트를 가진 문자와 그렇지 않은 문자를 동일 문자로 판단할지 여부를 나타낸다.
  
- `uf8mb4_bin`
  - 첫 번째 파트: 문자 집합의 이름
  - 두 번쨰 파트: 항상 bin 이라는 키워드가 사용된다. bin은 이진 데이터를 의미하며 이진 데이터로 관리되는 문자열 칼럼은 별도의 콜레이션을 가지지 않는다.

테이블의 구조는 `SHOW CREATE TABLE` 명령으로 확인할 수 있다. 그런데 칼럼이 디폴트 문자 집합이나 콜레이션을 사용할 대는 별도로 표시해주지 않으므로 확인하기 어렵다. 이때는 information_schema DB의 columns 뷰를 확인해 보면 된다.

### utf8mb4 문자 집합 콜레이션

`utf8_unicode_ci` 처럼 별도의 숫자 값이 명시되어 있지 않는 콜레이션이 있고, `utf8mb4_0900_ai_ci` 처럼 숫자 값이 명시되어 있는 콜레이션이 있는데 
이 숫자는 콜레이션의 알고리즘 버전을 의미한다. 별도의 숫자가 명시되어 있지 않은 경우 UCA 4.0.0 버전을 의미한다.
  
`utf8mb4_0900_ai_ci`와 같이 언어 비종속적인 콜레이션은 문자 집합의 기본 정렬 순서에 맞게 정렬 및 비교가 수행되며, 언어 종속적인 콜레이션은 
해당 언어에서 정의한 정렬 순서에 의해 정렬 및 비교가 수행된다. 그래서 특정 언어에 종속적인 정렬 순서가 필요하다면 그 언어에 맞는 콜레이션을 사용해야 한다.`(utf8mb4_ja_0900_as_cs)` 하지만 범용 응용 프로그램이면 `utf8mb4_0900_ai_ci`만으로도 충분하다.
  
UCA 9.0.0 버전은 그 이전 버전의 콜레이션보다 빠르다고 MySQL 메뉴얼에서 소개하고 있지만 실제로 테스트를 해보면 그렇지 않다. 또한 UCA 9.0.0 버전을
사용하는 콜레이션은 모두 `NO PAD` 옵션으로 문자열 비교 작업이 처리된다는 특징이 있다. 그렇다고 단순히 그 이전 버전의 UCA 4.0.0 콜레이션을 선택하는 것은 적절하지 않으며 콜레이션의 필요에 따라 결정해야 하는 문제이다. 
  
UCA 9.0.0 콜레이션은 최신 정렬 순서를 반영하고 있고, 문자열 뒤에 존재하는 공백도 유효 문자로 취급되어 비교되므로 주의해야 한다. 이전 콜레이션 버전을 사용하고 있는 프로그램에서 문자 집합과 콜레이션을 변경하는 작업은 위험할 수 있다. 새로운 서비스를 개발하고 있다면 성능과 관계없이 UCA 9.0.0 기반의 콜레이션을 사용하는 것이 바람직하다.
  
## 숫자

숫자를 저장하는 타입은 값의 정확도에 따라 크게 참값(Exact value)과 근삿값 타입으로 나눌 수 있다.

- 참값은 소수점 이하 값의 유무와 관계없이 그 값을 그대로 유지하는 것을 의미한다. 참값을 관리하는 데이터 타입으로는 INTEGER를 포함해 INT로 끝나는 타입과 DECIMAL이 있다.
- 근삿값은 흔히 부동 소수점이라고 불리는 값을 의미하며, 처음 칼럼에 저장한 값과 조화된 값이 정확하게 일치하지 않고 최대한 비슷한 값으로 관리하는 것을 의미한다. 근삿값을 관리하는 타입으로는 FLOAT와 DOUBLE이 있다.
  
DBMS에서는 근삿값은 저장할 때와 조회할 때의 값이 정확히 일치하지 않고, 유효 자릿수를 넘어서는 소수점 이하의 값은 계속 바뀔 수 있다. 특히 STATEMENT 포맷을
사용하는 복제에서는 소스 서버와 레플리카 서버 간 데이터 차이가 발생할 수도 있다. 
  
MySQL에서 FLOAT나 DOUBLE과 같은 부동 소수점 타입은 잘 사용하지 않는다. 또한 십진 표기법을 사용하는 DECIMAL 타입은 이진 표기법을 사용하는 타입보다 저장 공간을 
2배 이상 필요로 한다. 매우 큰 숫자 값이나 고정 소수점을 저장해야 하는 것이 아니라면 일반적으로 INTEGER나 BIGINT 타입을 자주 사용한다.

### 정수

- TINYINT
- SMALLINT
- MEDIUMINT
- INT
- BIGINT

DECIMAL 타입을 제외하고 정수를 저장하는 데 사용할 수 있는 데이터 타입으로는 5가지가 있다. 저장 가능한 숫자 값의 범위만 다를 뿐 다른 차이는 거의 없다.
입력이 가능한 수의 범위 내에서 최대한 저장 공간을 적게 사용하는 타입을 선택하면 된다.

### 부동 소수점

MySQL에서는 부동 소숮머을 저장하기 위해 FLOAT와 DOUBLE 타입을 사용할 수 있다. 부동 소수점을 사용하면 유효 소수점 값을 식별하기 어렵고 그 값을 따져서
크다 작다 비교를 하기가 쉽지 않은 편이다. 부동 소수점은 근삿값을 저장하는 방식이기 때문에 동등 비교는 사용할 수 없다. 이 밖에도 MySQL 매뉴얼을 살펴보면
부동 소수점을 사용할 때 주의할 내용이 많으므로 사용하기 전에 반드시 확인하는 것이 좋다.
  
복제에 참여하는 MySQL 서버에서 부동 소수점을 사용할 때는 특별히 주의해야 한다. 부동 소수점 타입의 데이터는 MySQL 서버의 바이너리 로그 포맷이
STATEMENT 타입인 경우 복세에서 소스 서버와 레플리카 서버 간의 데이터가 달라질 수 있다.
  
부동 소수점 값을 저장해야 한다면 유효 소수점의 자릿수만큼 10을 곱해서 정수로 만들어 그 값을 정수 타입의 칼럼에 저장하는 방법도 생각해볼 수 있다.

### DECIMAL

부동 소수점에서 유효 범위 이외의 값은 가변적이므로 정확한 값을 보장할 수 없다. 즉, 금액이나 대출이자 등과 같이 고정된 소수점까지 정확하게 관리해야 할 때는
FLOAT나 DOUBLE 타입을 사용해서는 안 된다. 그래서 소수점 위치가 가변적이지 않은 고정 소수점 타입을 위해 DECIMAL 타입을 제공한다.
  
DECIMAL 타입은 숫자 하나를 저장하는 데 1/2 바이트가 필요하므로 한 자리나 두 자릿수를 저장하는 데 1바이트가 필요하고 세 자리나 네 자리 숫자를 저장하는 데는 2바이트가 필요하다. 
  
그리고 DECIMAL 타입과 BIGINT 타입의 값을 곱하는 연산을 간단히 테스트해보면 미세하게 BIGINT 타입이 더 빠르다는 사실을 알 수 있고, 소수가 아닌 정숫값을 관리하기 위해 DECIMAL을 사용하는 건 성능상으로나 공간 사용면에서 좋지 않다. 단순히 정수를 관리하고자 한다면 INTEGER나 BIGINT를 사용하는 것이 좋다.

### 정수 타입의 칼럼을 생성할 때의 주의사항

부동 소수점이나 DECIMAL 타입을 이용해 칼럼을 정의할 때는 타입의 이름 뒤에 괄호로 정밀도를 표시하는 것이 일반적이다. 예를 들어
DECIMAL(20, 5)라고 정의하면 정수부를 15(=20-5)자리까지, 소수부를 5자리까지 저장할 수 있는 DECIMAL 타입을 생성한다. 그리고 DECIMAL(20)이라고 정의하는 경우
소수부 없이 20자리까지 저장할 수 있는 타입의 칼럼을 생성한다. FLOAT나 DOUBLE타입은 저장 공간의 크기가 고정적이므로 정밀도를 조절한다고 해서 저장 공간의 크기가 바뀌지는 않는다.
  
### 자동 증가(AUTO_INCREMENT) 옵션을 사용

테이블의 PK 를 구성하는 칼럼의 크기가 너무 크거나 PK로 사용할 만한 칼럼이 없을 때는 숫자 타입 칼럼에 AUTO_INCREMENT 옵션을 사용해 인조 키를 생성할 수 있다.
MySQL 서버의 시스템 설정을 이용해 AUTO_INCREMENT 칼럼의 자동 증가값이 얼마가 될지 변경할 수 있다.
  
AUTO_INCREMENT 옵션을 사용한 칼럼은 반드시 그 테이블에서 PK나 유니크 키의 일부로 정의해야 한다. 그런데 PK나 유니크 키가 여러 개의 칼럼으로 구성되면
AUTO_INCREMENT 속성의 칼럼값이 증가하는 패턴이 MyISAM과 InnoDB 스토리지 엔진에서 각각 달라진다.

- MyISAM 스토리지 엔진을 사용하는 테이블에서는 AUTO_INCREMENT 칼럼이 PK나 유니크 키의 아무 위치에서나 사용될 수 있다.
- InnoDB 스토리지 엔진을 사용하는 테이블에서는 AUTO_INCREMENT 칼럼으로 시작되는 인덱스(PK나 일반 인덱스)를 생성해야 한다. 즉 AUTO_INCREMENT 칼럼을 PK의 뒤쪽에 배치하면 오류가 발생한다.

AUTO_INCREMENT 칼럼은 테이블당 하나만 사용할 수 있다.

### 날짜와 시간

MySQL 5.6버전부터 TIME 타입과 DATETIME, TIMESTAMP 타입은 밀리초 단위의 데이터를 저장할 수 있게 됐다. 그래서 칼럼의 저장 공간 크기는 밀리초 단위를
몇 자리까지 저장하느냐에 따라 달라진다. 밀리초 단위는 2자리당 1바이트씩 공간이 더 필요하다. 그래서 MySQL 8.0에서는 마이크로초까지 저장 가능한 DATETIME(6)
타입은 8바이트(5바이트+3바이트)를 사용한다.

MySQL의 날짜 타입은 칼럼 자체에 타임존 정보가 저장되지 않으므로 DATETIME이나 DATE타입은 현재 DBMS 커넥션의 타임존과 관계없이 클라이언트로부터 입력된 값을 
그대로 저장하고 조회할 때도 변환없이 그대로 출력된다. 하지만 TIMESTAMP는 항상 UTC 타임존으로 저장되므로 타임존이 달라져도 값이 자동으로 보정된다.
  
그런데 MySQL 서버에서 칼럼 타입이 TIMESTAMP든 DATETIME이든 관계없이, JDBC 드라이버는 날짜 및 시간 정보를 MySQL 타임존에서 JVM의 타임존으로 변환해서 출력한다. ORM을 사용할 때는 ORM이 코드 내부적으로 값을 자동으로 페치해서 응용 프로그램 코드로 변환하기 때문에 DATETIME 타입의 칼럼값을 어떤 JDBC API를 이용해서 페치하는지 테스트하는 것이 좋다.

### ENUM

ENUM 타입의 가장 큰 단점은 칼럼에 저장되는 문자열 값이 테이블의 구조가 되면서 기본 ENUM 타입에 새로운 값을 추가해야 한다면 테이블의 구조를 변경해야 한다.
예전 MySQL 서버에서 ENUM 타입의 아이템이 새로 추가되면 테이블을 항상 리빌드해야 했다. 이러한 문제로 인해 MySQL 서버에서 ENUM 타입은 별로 사용되지 않았다.
하지만 5.6버전부터는 새로 추가되는 아이템이 ENUM 타입의 `제일 마지막으로 추가되는 형태`라면 테이블의 구조(메타데이터) 변경만으로 즉시 완료된다.
  
마지막에 새로운 아이템을 새로 추가하는 작업은 INSTANT 알고리즘으로 메타데이터 변경만으로 완료된다는 것을 알 수 있지만, 기존 ENUM 타입의 아이템들이 순서가 변경되거나 새로운 아이템이 추가되는 경우에는 COPY 알고리즘에 읽기 잠금까지 필요하다. 테이블이 매우 크다면 가독성이 좀 떨어지더라도 새로운 아이템을 ENUM 타입의 마지막에 추가하는 것이 MySQL 서버의 가용성을 높이는 방법이다.

ENUM 타입은 테이블 구조에 정의된 코드 값만 사용할 수 있게 강제한다는 장점도 있지만, 더 큰 장점은 데이터베이스 서버의 디스크 저장 공간의 크기를 줄여준다는 것이다. 테이블 레코드 건수가 많지 않다면 디스크 사용량은 큰 장점이 아닐 수도 있다. 하지만 레코드가 억 단위를 넘어간다면 10글자\~20글자를 넘어서는 문자열 칼럼보다 ENUM 타입이 매우 작은 1~2GB 저장 공간을 줄일 수 있다. 이 칼럼이 여러 개의 인덱스에 사용된다면 용량을 몇 배로 줄이는 효과를 얻을 수 있다.
  
디스크의 데이터는 InnoDB 버퍼 풀로 적재돼야 쿼리에서 비로소 사용될 수 있기 때문에, 디스크 데이터가 크다는 것은 메모리도 그만큼 많이 필요해진다는 이야기다. 그만큼 ENUM은 장점이 매우 크고, 메모리 사용량 절감 효과를 빼더라도 디스크의 사용량이 적다면 백업이나 복구 시간을 줄일 수 있기 때문에 장점이 된다. 당장 장애가 발생했는데 백업 파일을 복사하는데 3~4시간이 걸린다면 이 시간동안은 서비스가 불가능해진다는 것이다. ENUM 타입을 더나서라도 가능하면 디스크의 데이터 파일 크기를 줄이는 것이 성능과 운영에 많은 도움이 된다.

### SET

SET 타입도 테이블의 구조에 정의된 아이템을 정숫값으로 매핑해서 저장하는 방식은 ENUM과 똑같다. 그런데 SET은 하나의 칼럼에 1개 이상의 값을 저장할 수 있다.
여러 개의 값을 저장할 수는 있지만 실제 여러 개의 값을 저장하는 공간을 가지는 것은 아니다. SET 타입은 아이템의 값의 멤버 수가 8개 이하이면 1바이트의 저장 공간을 사용하며 9개에서 16개 이하이면 2바이트를 사용하고 똑같은 방식으로 최대 8바이트까지 저장 공간을 사용한다.

### TEXT와 BLOB

MySQL에서 대량 데이터를 저장하려면 TEXT나 BLOB 타입을 사용해야 하는데, 이 두 타입은 많은 부분에서 거의 똑같은 설정이나 방식으로 작동한다. 거의 유일한 차이점은 TEXT 타입은 문자열을 저장하는 대용량 칼럼이라서 문자 집합이나 콜레이션을 가진다는 것이고, BLOB 타입은 이진 데이터 타입이라서 별도의 문자 집합이나 콜레이션을 가지지 않는다는 것이다.

TEXT나 BLOB 타입은 주로 다음과 같은 상황에서 사용하는 것이 좋다.

- 칼럼 하나에 저장되는 문자열이나 이진 값의 길이가 예측할 수 없이 클 때 TEXT나 BLOB을 사용한다. 하지만 다른 DBMS와는 달리 MySQL 에서는 값의 크기가 4000바이트를 넘을 때 반드시 BLOB이나 TEXT를 사용해야 하는 것은 아니다. MySQL에서는 레코드의 전체 크기가 64KB를 넘지 않는 한도 내에서는 VARCHAR나 VARBINARY의 길이는 제한이 없다. 그래서 용도에 따라서는 4000 바이트 이상의 값을 저장하는 칼럼도 VARCHAR나 VARBINARY 타입을 이용할 수 있다.
- MySQL에서는 버전에 따라 조금씩 차이는 있지만 일반적으로 하나의 레코드는 전체 크기가 64KB를 넘어설 수 없다. VARCHAR나 VARBINARY와 같은 가변 길이 칼럼은 최대 저장 가능 크기를 포함해 64KB로 크기가 제한된다. 레코드의 전체 크기가 64KB를 넘어서서 더 큰 칼럼을 추가할 수 없다면 일부 칼럼을 TEXT나 BLOB 타입으로 전환해야 할 수도 있다.

### 가상 칼럼(파생 칼럼)

MySQL 서버의 가상 칼럼은 크게 가상 칼럼과 스토어드 칼럼으로 구분할 수 있다.

```SQL
CREATE TABLE tb_virtual_column {
  id INT NOT NULL AUTO_INCREMENT,
  price DECIMAL(10, 2) NOT NULL DEFAULT '0.00',
  quantity INT NOT NULL DEFAULT 1,
  total_price DECIMAL(10, 2) AS (quantity * price) VIRTUAL,
  PRIMARY KEY (id)
);
```
```SQL
CREATE TABLE tb_stored_column {
  id INT NOT NULL AUTO_INCREMENT,
  price DECIMAL(10, 2) NOT NULL DEFAULT '0.00',
  quantity INT NOT NULL DEFAULT 1,
  total_price DECIMAL(10, 2) AS (quantity * price) STORED,
  PRIMARY KET (id)
);
```

가상 칼럼과 스토어드 칼럼 모두 칼럼 정의 뒤에 AS 절로 계산식을 정의한다. 이때 마지막에 STORED 키워드가 사용되면 스토어드 칼럼으로 생성되며, 그 이외의 경우에는 항상 가상 칼럼으로 정의된다. 즉 아무런 키워드도 정의되지 않으면 MySQL 서버는 기본 모드인 VIRTUAL로 칼럼을 생성한다. 가상 칼럼은 다른 칼럼의 값을 참조해서 계산된 값을 관리하기 때문에 항상 AS 절 뒤에는 계산식이나 데이터 가공을 위한 표현식을 정의한다.
  
- 가상 칼럼
  - `칼럼의 값이 디스크에 저장되지 않음`
  - 칼럼의 구조 변경은 테이블 리빌드를 필요로 하지 않음
  - 칼럼의 값은 레코드가 읽히기 전 또는 BEFORE 트리거 실행 직후에 계산되어 만들어짐
- 스토어드 칼럼
  - `칼럼의 값이 물리적으로 디스크에 저장됨`
  - 칼럼의 구조 변경은 다른 일반 테이블과 같이 필요 시 테이블 리빌드 방식으로 처리 됨
  - INSERT와 UPDATE 시점에만 칼럼의 값이 계산됨
 
가상 칼럼에 인덱스를 생성하게 되면 테이블의 레코드는 가상 칼럼을 포함하지 않지만 해당 인덱스는 계산된 값을 저장하기 때문에, 인덱스가 생성된 가상 칼럼의 경우 변경이 필요하다면 인덱스 리빌드 작업이 필요할 수 있다. 가상 칼럼과 스토어드 칼럼 중 어떤 것을 선택해야 할지는 각 차이점을 이용해 어렵지 않게 판단할 수 있다. 가상 칼럼은 데이터를 조회하는 시점에 매번 계산되기 때문에 가상 칼럼의 값을 계산하는 과정이 복잡하고 시간이 오래 걸린다면 스토어드 칼럼으로 변경하는 것이 성능 향상에 도움이 될 수 있다. 반대로 계산 과정이 빠른 반면 상대적으로 결과가 많은 저장 공간을 차지한다면 가상 칼럼을 선택하는 것이 저장 공간의 절약과 메모리의 효율을 높일 수 있다.

## Reference 

**위 글은 책 RealMySQL 8.0 2권을 구입하여 읽고 정리한 내용입니다.**
- [도서 홈페이지 https://wikibook.co.kr/realmysql802/](https://wikibook.co.kr/realmysql802/)
