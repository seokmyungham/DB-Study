파티션 기능은 테이블을 **논리적으로는 하나의 테이블**이지만 **물리적으로는 여러 개의 테이블로 분리**해서 관리할 수 있게 해준다. 파티션 기능은 **주로 대용량의 테이블을 물리적으로 여러 개의 소규모 테이블로 분산하는 목적으로 사용**한다. 

하지만 파티션 기능은 대용량 테이블에 사용하면 무조건 성능이 빨라지는 만병 통치약이 아니다. 어떤 쿼리를 사용하느냐에 따라 오히려 성능이 더 나빠지는 경우도 자주 발생할 수 있다. MySQL 서버는 다양한 파티션 방법을 제공하지만 여기서는 자주 사용하는 파티션 방법과 사용 시 주의해야할 사항 위주로 살펴보겠다. 

# MySQL 파티션
MySQL 파티션이 적용된 테이블에서 **INSERT나 SELECT 등과 같은 쿼리가 어떻게 실행**되는지 이해한다면 파티션을 어떻게 사용하는 것이 가장 최적일지 쉽게 이해할 수 있을 것이다. 파티션이 SQL 문장을 수행하는데 어떻게 영향을 미치는지, 그리고 파티션으로 기대할 수 있는 장점으로 무엇이 있는지 살펴보자. 

## 파티션을 사용하는 이유 
테이블의 데이터가 많아진다고 해서 무조건 파티션을 적용하는 것이 효율적인 것은 아니다. **하나의 테이블이 너무 커서 인덱스의 크기가 물리적인 메모리보다 훨씬 크거나** **데이터 특성상 주기적인 삭제 작업**이 필요한 경우 등이 파티션이 필요한 대표적인 예라고 할 수 있다. 각 경우에 대해 지금부터 하나씩 자세히 살펴보자.

### 1. 단일 INSERT와 단일 또는 범위 SELECT의 빠른 처리 
데이터베이스에서 인덱스는 일반적으로 SELECT를 위한 것으로 보이지만 UPDATE나 DELETE 쿼리를 위해 필요한 때도 많다. 물론 레코드를 변경하는 쿼리를 실행하면 인덱스의 변경을 위한 부가적인 작업이 발생하긴 하지만 UPDATE나 DELETE 처리를 위해 대상 레코드를 검색하려면 인덱스가 필수적이다. 하지만 인덱스가 커지면 커질수록 SELECT는 말할 것도 없고, INSERT나 UPDATE, DELETE 작업도 함께 느려지는 단점이 있다. 

특히 한 테이블의 인덱스 크기가 물리적으로 MySQL이 사용 가능한 메모리 공간보다 크다면 그 영향은 더 심각할 것이다. 테이블의 데이터는 실질적인 물리 메모리보다 큰 것이 일반적이겠지만 인덱스의 워킹 셋(Working Set)이 실질적인 물리 메모리보다 크다면 쿼리 처리가 상당히 느려질 것이다.


큰 테이블을 파티션하지 않고 그냥 사용할 때와 작은 파티션으로 나눠서 워킹 셋의 크기를 줄였을 때 인덱스의 워킹 셋이 물리적인 메모리를 어떻게 사용하는지를 보여준다.


![](https://velog.velcdn.com/images/chocochip/post/03f79e73-05b1-4eba-9c1e-f76e43de9da0/image.png)



![](https://velog.velcdn.com/images/chocochip/post/40f39df4-fd69-4b8c-8203-82f5c957a74f/image.png)

파티션하지 않고 하나의 큰 테이블로 사용하면 인덱스도 커지고 그만큼 물리적인 메모리 공간도 많이 필요해진다는 사실을 알 수 있다. 결과적으로 파티션은 데이터와 인덱스를 조각화해서 물리적 메모리를 효율적으로 사용할 수 있게 만들어준다. 

테이블의 데이터가 10GB이고 인덱스가 3GB라고 가정해보자. 하지만 대부분의 테이블은 13GB 전체를 항상 사용하는 것이 아니라 그중에서 일정 부분만 활발하게 사용한다. 즉, 게시물이 100만 건이 저장된 테이블이라고 하더라도 **그중에서 최신 20~30%의 게시물만 활발하게 조회될 것이다. 대부분의 테이블 데이터가 이런 형태로 사용된다고 볼 수 있는데, 활발하게 사용되는 데이터를 워킹 셋(Working Set)이라고 표현**한다. 테이블의 데이터를 활발하게 사용되는 워킹 셋과 그렇지 않은 부분으로 나눠서 파티션할 수 있다면 상당히 효과적으로 성능을 개선할 수 있을 것이다. 

### 2. 데이터의 물리적인 저장소를 분리 
데이터 파일이나 인덱스 파일이 파일 시스템에서 차지하는 공간이 크다면 그만큼 백업이나 관리 작업이 어려워진다. 더욱이 테이블의 데이터나 인덱스를 파일 단위로 관리하는 MySQL에서 더 치명적인 문제가 될 수도 있다. 이러한 문제는 파티션을 통해 **파일의 크기를 조절하거나 파티션별 파일들이 저장될 위치나 디스크를 구분해서 지정해 해결하는 것도 가능**하다. 하지만 MySQL에서는 테이블의 파티션 단위로 인덱스를 생성하거나 파티션별로 다른 인덱스를 가지는 형태는 지원하지 않는다. 

### 3. 이력 데이터의 효율적인 관리
요즘은 거의 모든 애플리케이션이 로그라는 이력 데이터를 가지고 있는데, 이는 단기간에 대량으로 누적됨과 동시에 일정 기간이 지나면 쓸모가 없어진다. 로그 데이터는 결국 시간이 지나면 별도로 아카이빙하거나 백업한 후 삭제해버리는 것이 일반적이며, 특히 다른 데이터에 비해 라이프 사이클이 상당히 짧은 것이 특징이다. 로그 테이블에서 불필요해진 데이터를 백업하거나 삭제하는 작업은 일반 테이블에서는 상당히 고부하의 작업에 속한다. 

하지만 **로그 테이블을 파티션 테이블로 관리한다면 불필요한 데이터 삭제 작업은 단순히 파티션을 추가하거나 삭제하는 방식으로 간단하고 빠르게 해결할 수 있다.** 대량의 데이터가 저장된 로그 테이블을 기간 단위로 삭제한다면 MySQL 서버에 전체적으로 미치는 부하뿐만 아니라 로그 테이블 자체의 동시성에도 영향을 미칠 수 있다. 하지만 파티션을 이용하면 이러한 문제를 대폭 줄일 수 있다. 
 
## MySQL 파티션의 내부 처리 
파티션이 적용된 테이블에서 레코드의 INSERT, UPDATE, SELECT가 어떻게 처리되는지 확인해보기 위해 간단한 테이블을 가정해 보자. 
```sql
mysql> CREATE TABLE tb_article ( 
	article_id INT NOT NULL, 
    reg_date DATETIME NOT NULL,
    ... 
    PRIMARY KEY(article_id, reg_date) 
    ) PARTITION BY RANGE (YEAR(reg_date) ) ( 
    	PARTITION p2009 VALUES LESS THAN (2010), 
        PARTITION p2010 VALUES LESS THAN (2011), 
        PARTITION p2011 VALUES LESS THAN (2012), 
        PARTITION p9999 VALUES LESS THAN MAXVALUE 
    ); 
```
여기서 게시물의 **등록 일자(reg_date)에서 연도 부분은 파티션 키**로서 해당 레코드가 어느 파티션에 저장될지를 결정하는 중요한 역할을 담당한다. 이제 tb_article 테이블에 대해 INSERT, UPDATE, SELECT 같은 쿼리가 어떻게 처리되는지 하나씩 살펴보자. 

### 파티션 테이블의 레코드 INSERT 
INSERT 쿼리가 실행되면 MySQL 서버는 INSERT되는 칼럼의 값 중에서 파티션 키 값을 이용해 파티션 표현식을 평가하고, 그 결과를 이용해 레코드가 저장될 **적절한 파티션을 결정**한다. 새로 INSERT되는 레코드를 위한 파티션이 결정되면 나머지 과정은 파티션되지 않은 일반 테이블과 동일하게 처리된다.

### 파티션 테이블의 UPDATE 
UPDATE 쿼리를 실행하려면 변경 대상 레코드가 어느 파티션에 저장돼 있는지 찾아야 한다. 이때 UPDATE 쿼리의 **WHERE 조건에 파티션 키 칼럼이 조건으로 존재한다면 그 값을 이용해 레코드가 저장된 파티션에서 빠르게 대상 레코드를 검색**할 수 있다. 하지만 **WHERE 조건에 파티션 키 칼럼의 조건이 명시되지 않았다면 MySQL 서버는 변경 대상 레코드를 찾기 위해 테이블의 모든 파티션을 검색**해야 한다. 그리고 실제 레코드를 변경하는 작업의 절차는 UPDATE 쿼리가 어떤 칼럼의 값을 변경하느냐에 따라 큰 차이가 생긴다.

- 파티션 키 이외의 칼럼만 변경될 때: 파티션이 적용되지 않은 일반 테이블과 마찬가지로 칼럼 값만 변경
- 파티션 키 칼럼이 변경될 때
  1. 기존의 레코드가 저장된 파티션에서 해당 레코드를 삭제
  2. 변경되는 파티션 키 칼럼의 표현식을 평가하고, 그 결과를 이용해 레코드를 이동시킬 새로운 파티션을 결정해서 레코드를 새로 저장

### 파티션 테이블의 검색 
SQL이 수행되기 위해 파티션 테이블을 검색할 때 성능에 크게 영향을 미치는 조건은 다음과 같다. 
- WHERE 절의 조건으로 검색해야 할 **파티션을 선택할 수 있는가?** 
- WHERE 절의 조건이 인덱스를 효율적으로 사용(인덱스 레인지 스캔)할 수 있는가? 

두 번째 내용은 파티션 테이블뿐만 아니라 파티션되지 않은 일반 테이블의 검색 성능에도 똑같이 영향을 미친다. 하지만 파티션 테이블에서는 첫 번째 선택사항의 결과에 의해 두 번째 선택사항의 작업 내용이 달라질 수 있다. 위 두 가지 주요 선택사항의 각 조합이 어떻게 실행되는지 한번 살펴보자. 

- 파티션 선택 가능 + 인덱스 효율적 사용 가능: 두 선택사항이 모두 사용 가능할 때 쿼리가 가장 효율적으로 처리될 수 있다. 이때는 파티션의 개수와 관계없이 검색을 위해 꼭 필요한 파티션의 인덱스만 레인지 스캔한다. 

- 파티션 선택 불가 + 인덱스 효율적 사용 가능: WHERE 조건에 일치하는 레코드가 저장된 파티션을 걸러낼 수 없기 때문에 우선 **테이블의 모든 파티션을 대상으로 검색**해야 한다. 하지만 각 파티션에 대해서는 인덱스 레인지 스캔을 사용할 수 있기 때문에 최종적으로 테이블에 존재하는 모든 파티션의 개수만큼 인덱스 레인지 스캔을 수행해서 검색 하게 된다. 이 작업은 파티션 개수만큼의 테이블에 대해 인덱스 레인지 스캔을 한 다음, **결과를 병합해서 가져오는 것**과 같다. 

- 파티션 선택 가능 + 인덱스 효율적 사용 불가: 검색하려는 레코드가 저장된 파티션을 선별할 수 있기 때문에 파티션 개수와 관계없이 검색을 위해 **필요한 파티션만 읽으면 된다**. 하지만 인덱스는 이용할 수 없기 때문에 대상 파티션에 대해 풀 테이블 스캔을 한다. 이는 각 파**티션의 레코드 건수가 많다면 상당히 느리게 처리**될 것이다. 

- 파티션 선택 불가 + 인덱스 효율적 사용 불가: WHERE 조건에 일치하는 파티션을 선택할 수가 없기 때문에 테이블의 모든 파티션을 검색해야 한다. 하지만 각 파티션을 검색하는 작업 자체도 인덱스 레인지 스캔을 사용할 수 없기 때 문에 풀 테이블 스캔을 수행해야 한다. 

앞에서 살펴본 선택사항의 4가지 조합 가운데 마지막 **세 번째와 네 번째 방식은 가능하다면 피하는 것이 좋다.** 그리고 두 번째 조합 또한 하나의 테이블에 파티션의 개수가 많을 때는 MySQL 서버의 부하도 높아지고 처리 시간도 많이 느려지므로 주의하자.

### 파티션 테이블의 인덱스 스캔과 정렬 

MySQL의 파티션 테이블에서 인덱스는 **전부 로컬 인덱스에 해당**한다. 즉, 모든 인덱스는 파티션 단위로 생성되며, 파티션과 관계없이 테이블 전체 단위로 글로벌하게 하나의 통합된 인덱스는 지원하지 않는다는 것을 의미한다. 



파티션의 순서는 정렬이 보장되지 않는다. 즉, 파티션되지 않은 테이블에서는 인덱스를 순서대로 읽으면 그 칼럼으로 정렬된 결과를 바로 얻을 수 있지만 파티션된 테이블에서는 그렇지 않다. 

실행 계획을 확인해보면 Extra 칼럼에 별도의 정렬 작업을 의미하는 "Using filesort" 코멘트가 표시되지 않는다. 쿼리의 실행 계획에서도 별도의 정렬을 수행했다는 메시지는 표시되지 않는다. 


실제 MySQL 서버는 여러 파티션에 대해 인덱스 스캔을 수행할 때 각 파티션으로부터 조건에 일치하는 레코드를 정렬된 순서대로 읽으면서 우선순위 큐에 임시로 저장한다. 그리고 우선순위 큐에서 다시 필요한 순서(인덱스의 정렬 순서)대로 데이터를 가져가는 것이다. 이는 각 파티션에서 읽은 데이터가 이미 정렬돼 있는 상태라서 가능한 방법이다. 결론적으로 파티션 테이블에서 인덱스 스캔을 통해 레코드를 읽을 때 MySQL 서버가 별도의 정렬 작업을 수행하지는 않는다. 하지만 일반 테이블의 인덱스 스캔처럼 결과를 바로 반환하는 것이 아니라 내부적으로 큐 처리가 한 번 필요한 것이다. 

### 파티션 프루닝 
옵티마이저에 의해 3개의 파티션 가운데 2개만 읽어도 된다고 판단되면 불필요한 파티션에는 전혀 접근하지 않는다. 이렇게 **최적화 단계에서 필요한 파티션만 골라내고 불필요한 것들은 실행 계획에서 배제하는 것을 파티션 프루닝**이라고 한다. 이러한 파티션 프루닝 정보는 실행 계획을 확인해보면 옵티마이저가 어떤 파티션만 접근하는지 알 수 있다. EXPLAIN 명령의 결과에 보이는 partitions 칼럼을 살펴보면 쿼리가 어떤 파티션만 조회하는지 확인할 수 있다.


# 주의사항 

## 파티션의 제약 사항 
우선 MySQL 서버의 파티션 기능의 제약 사항을 이해하려면 먼저 용어 몇 가지를 이해해야 한다. 다음과 같은 파티션 테이블을 가정해보자.
```sql
mysql> 	CREATE TABLE tb_article ( 
		article_id INT NOT NULL AUTO_INCREMENT, 
        reg_date DATETIME NOT NULL,  
        ...
        PRIMARY KEY(article_id, reg_date) 
        ) PARTITION BY RANGE (YEAR(reg_date) ) ( 
        	PARTITION p2009 VALUES LESS THAN (2010), 
            PARTITION p2010 VALUES LESS THAN (2011), 
            PARTITION p2011 VALUES LESS THAN (2012), 
            PARTITION p9999 VALUES LESS THAN MAXVALUE
        );
```

이 테이블 정의에서 "PARITITION BY RANGE" 절은 이 테이블이 레인지 파티션을 사용한다는 것을 의미한다. 그리고 파티션 칼럼은 reg_date이며, 파티션 표현식으로는 "YEAR(reg_date)"가 사용됐다. 즉 th_article 테이블은 reg_date 칼럼에서 YEAR()라는 MySQL 내장 함수를 이용해 연도만 추출하고, 그 연도를 이용해 테이블을 연도 범위별로 파티션하고 있다. 

이제 MySQL 서버의 파티션이 가지는 제약 사항들을 살펴보자. 
- 스토어드 루틴이나 UDF, 사용자 변수 등을 파티션 표현식에 사용할 수 없다. 
- 파티션 표현식은 일반적으로 칼럼 그 자체 또는 MySQL 내장 함수를 사용할 수 있는데, 여기서 일부 함수들은 파티 선 생성은 가능하지만 파티션 프루닝을 지원하지 않을 수도 있다. 
- 프라이머리 키를 포함해서 테이블의 모든 유니크 인덱스는 파티션 키 칼럼을 포함해야 한다. 
- 파티션된 테이블의 인덱스는 **모두 로컬 인덱스**이며, 동일 테이블에 소속된 모든 파티션은 같은 구조의 인덱스만 가질 수 있다. 또한 파티션 개별로 인덱스를 변경하거나 추가할 수 없다. 
- 동일 테이블에 속한 모든 파티션은 동일 스토리지 엔진만 가질 수 있다. 
- 최대(서브 파티션까지 포함해서) 8192개의 파티션을 가질 수 있다. 
- 파티션 생성 이후 MySQL 서버의 sql_mode 시스템 변수 변경은 데이터 파티션의 일관성을 깨뜨릴 수 있다. 파티션 테이블에서는 외래키를 사용할 수 없다. 
- 파티션 테이블은 전문 검색 인덱스 생성이나 전문 검색 쿼리를 사용할 수 없다. 
- 공간 데이터를 저장하는 칼럼 타입(POINT. GEOMETRY, …)은 파티션 테이블에서 사용할 수 없다. 
- 임시 테이블(Temporary table)은 파티션 기능 사용할 수 없다. 


일반적으로 **파티션 테이블을 생성할 때 가장 크게 영향을 미치는 제약 사항은 모든 유니크 인덱스에 파티션 키 칼럼이 포함돼야 한다는 것**이다. tb_article 테이블의 article_id 칼럼은 AUTO INCREMENT를 사용하기 때문에 article_id 칼럼만으로 프라이머리 키를 사용해도 충분하다. 그런데 tb_article 테이블에서는 (article_id, reg_date) 칼럼의 조합으로 프라이머리 키를 선정했다. 이는 파티션 키로 사용되는 칼럼은 반드시 프라이머리 키 일부로 참여해야 한다는 제약 사항 때문이다. 실제 article_id만으로 유니크한 값을 가지기 때문에 reg_date 칼럼을 프라이머리 키 마지막에 추가하는 것은 아무런 의미가 없지만 reg_date 칼럼으로 파티션을 적용하기 위해서는 이 방법밖에 없다.

한 가지 더 주의해야할 사항은 MySQL 서버에서 파티션 포현식에는 기본적인 산술 연산자인 "+, -, ..."를 사용할 수 있으며, MySQL 내장함수도 사용할 수 있다.(ABS(), CEILING(), ...)

내장 함수들을 파티션 표현식에 사용할 수 있다고 해서 이 내장 함수들이 모두 파티션 프루닝 기능을 지원하는 것은 아니다. 파티션 테이블을 처음 설계할 때는 파티션 프루닝 기능이 정상적으로 작동하는지 확인한 후 응용 프로그램에 적용하는 것을 권장한다.

## 파티션 사용 시 주의사항

파티션 테이블의 경우 PK를 포함한 유니크 키에 대해서는 큰 제약 사항이 있다. 팥티션의 목적이 범위를 좁히는 것인데, 유니크 인덱스는 중복 레코드에 대한 체크 작업 때문에 범위가 좁혀지지 않는다. 또한 MySQL의 파티션은 일반 테이블과 같이 별도의 파일로 관리되는데, 이와 관련하여 MySQL 서버가 조작할 수 있는 파일의 개수와 연관이 있다.

### 파티션과 유니크 키(PK 포함)
종류와 관계없이 테이블에 유니크 인덱스(PK 포함)이 있으면 파티션 키는 모든 유니크 인덱스의 일부 또는 모든 칼럼을 포함해야 한다.

### 파티션과 open_files_limit 시스템 변수 설정
MySQL에서는 일반적으로 테이블을 파일 단위로 관리하기 때문에 MySQL 서버에서 동시에 오픈된 파일의 개수가 상당히 많아질 수 있다. 이를 제한하기 위해 open_files_limit 시스템 변수에 동시에 오픈할 수 있는 적절한 개수를 설정할 수 있다. 파티션되지 않은 일반 테이블은 테이블 1개당 오픈된 파일의 개수가 2~3개 수준이지만 파티션 테이블에서는 (파티션의 개수 * 2~3)개가 된다. 그래서 파티션을 사용하는 경우에는 open_files_limit 시스템 변수를 적절히 높은 값으로 설정해 줄 필요가 있다.

# MySQL 파티션의 종류
다른 DBMS와 마찬가지로 MySQL에서도 4가지 기본 파티션 기법을 제공하고 있으며, 해시와 키 파티션에 대해서는 Linear 파티션과 같은 추가적인 기법도 제공한다.

- 레인지 파티션
- 리스트 파티션
- 해시 파티션
- 키 파티션



## 레인지 파티션
파티션 키의 연속된 범위로 파티션을 정의하는 방법으로, 가장 일반적으로 사용되는 파티션 방법 중 하나다. 다른 파티션 방법과는 달리 MAXVALUE라는 키워드를 이용해 명시되지 않은 범위의 키 값이 담긴 레코드를 저장하는 파티션을 정의할 수 있다.

### 용도
아래의 성격을 지닌 테이블에서 레인지 파티션을 사용하는 것이 좋다. 
- 날짜를 기반으로 데이터가 누적되고 연도나 월, 또는 일 단위로 분석하고 삭제해야 할 때
- 범위 기반으로 데이터를 여러 파티션에 균등하게 나눌 수 있을 때
- 파티션 키 위주로 검색이 자주 실행될 때


데이터베이스에서 파티션의 장점은 2가지이다.

1. 큰 테이블을 작은 크기의 파티션으로 분리
2. 필요한 파티션만 접근(Read, Write)

위 2가지 장점 중 첫 번째보다 두 번째 장점의 효과가 큰 편이다. 그런데 문제는 파티션을 적용하면서 첫 번째 장점에만 집중하고 두 번째 장점은 취하지 못하고 경우가 있다. 결과적으로 파티션 때문에 오히려 MySQL 서버의 성능이 더 떨어트리게 되는 것이다. 실제 DB 서버의 파티션에서 이 두 가지 장점을 모두 취하기는 매우 어렵지만, 다행히 이력을 저장하는 테이블에서 레인지 파티션은 두 가지 장점을 모두 어렵지 않게 취할 수 있다.

> 여러 응용 프로그램에서 사용했던 파티션은 대부분 이력을 저장하는 로그 테이블에 레인지 파티션을 적용한 경우가 좋다.

## 리스트 파티션

리스트 파티션은 파티션 키 값 하나하나를 리스트로 나열한다. 
### 용도

- 파티션 키 값이 코드 값이나 카테고리와 같이 고정적일 때
- 키 값이 연속되지 않고 정렬 순서와 관계없이 파티션을 해야 할 때
- 파티션 키 값을 기준으로 레코드의 건수가 균일하고 검색 조건에 파티션 키가 자주 사용될 때


### 주의사항


## 해시 파티션

## 키 파티션