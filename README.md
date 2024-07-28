# 샘플 데이터베이스 다운로드

- 우선 데이터베이스 최적화 테스트를 해보기 전 테스트를 해볼 샘플 데이터베이스를 다운받아보겠습니다.
- 우선 https://dev.mysql.com/doc/index-other.html로 접속하면 Example Databases가 있는데 저는 이 중 employee data (large dataset, includes data and test/verification suite)를 사용하겠습니다!

<img width="703" alt="image" src="https://github.com/user-attachments/assets/90164005-6f89-42fa-9757-f2604d69dd87">

- Github를 클릭하여 깃허브로 들어가준 뒤 Code-Download ZIP으로 파일을 다운로드받아줍니다!

<img width="703" alt="image" src="https://github.com/user-attachments/assets/b8dc8f40-1645-4e91-b84b-c3db5561130c">

- 리드미에 적힌대로 다운로드한 파일 압축 해제 후 해당 경로로 들어가서 아래 명령어를 입력해줍니다.

```sql
mysql < employees.sql
```

- 하지만 저는 데이터베이스에 비밀번호가 걸려있어서 `ERROR 1045 (28000): Access denied for user '\'@'localhost' (using password: NO)`가 발생하였기 떄문에 로그인할 사용자를 지정하여 비밀번호를 입력하였습니다.

```sql
mysql -u root -p < employees.sql
```

<img width="353" alt="image" src="https://github.com/user-attachments/assets/19a60bb9-a60d-4fa8-b800-9f09b4460b4a">

- 다운로드 받은 파일을 테스트 파일을 통해 테스트해봅니다.

```sql
mysql -u root -p -t < test_employees_md5.sql
```

<img width="457" alt="image" src="https://github.com/user-attachments/assets/778839a9-bc2f-41b6-9268-753f6bb9078c">

# 데이터베이스 확인

- 이제 데이터베이스를 확인해보겠습니다!

```sql
// mysql 접속
mysql -u root -p

// 데이터베이스 목록 확인
show databases;

// 데이터베이스 접속
use employees;

// 테이블 확인
show tables;
```

<img width="483" alt="image" src="https://github.com/user-attachments/assets/4a417f50-7a34-4163-88c6-c16ad54d9732">

employee 테이블의 데이터를 확인해보겠습니다!

```sql
select * from employees;
```

<img width="590" alt="image" src="https://github.com/user-attachments/assets/fc33da69-8f33-41dd-946a-02462f5d3bbb">

- 300024개의 행이 있다고 합니다!
- 번호는 499999까지 있는데 데이터가 30만개라는게 이상해서 중간을 보니 10만대의 대부분과 30만대 전체가 없는 것을 확인하였습니다!

# MySQL Slow Query 설정

- MySQL의 Slow Query Logging을 사용하여 수행 시간이 오래 걸리는 로그를 남길 수 있습니다.
- Slow Query Logging을 설정하려면 my.cnf파일을 변경해야하는데 저는 아래 경로에 있었습니다!

`/opt/homebrew/etc/my.cnf`

```sql
// 본인의 my.cnf 경로로 이동 후
$ vim my.cnf

// 설정
// 슬로우 쿼리를 TABLE로 출력, FILE로도 설정 가능
// FILE로 설정했다면 slow_query_log_file로 출력 파일 위치 설정 가능
log_output = TABLE

// 슬로우 쿼리 활성화
slow_query_log = 1

// 아래 설정한 초(second) 이상 쿼리가 수행되면 슬로우 쿼리에 기록
long_query_time = 1
```

<img width="590" alt="image" src="https://github.com/user-attachments/assets/e5103e27-8deb-482b-963b-d9ca03e746d3">

- mysql 서버 재시작 후 slow_log 확인

```sql
mysql.server restart

select * from mysql.slow_log;
```

<img width="590" alt="image" src="https://github.com/user-attachments/assets/df65cdde-e5b7-41d6-bd47-558007b9d548">

# 테스트

- 이제 1초 이상 걸리는 쿼리를 작성하여 테스트를 진행해보겠습니다!
- 저는 총 2가지 상황을 가정하여 테스트해보겠습니다.

## 1. 서브쿼리를 포함하는 복잡한 쿼리 최적화

### 쿼리 실행

```sql
SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
WHERE s.salary = (
    SELECT MAX(salary)
    FROM salaries
    WHERE from_date > '2000-01-01'
)
AND e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC, s.salary DESC;
```

<img width="521" alt="image" src="https://github.com/user-attachments/assets/1eec2b63-bcb0-447a-ae83-714966662587">

- 검색에 2.98초가 걸리는 것을 확인할 수 있었습니다.

### 슬로우 쿼리를 이용한 확인

- 슬로우 쿼리를 이용하여 확인해보겠습니다.

```sql
select * from mysql.slow_log;
```

<img width="521" alt="image" src="https://github.com/user-attachments/assets/a105cbab-b303-41c7-884e-cb632dd21a5f">

- 이전 것들은 1초가 넘는 쿼리를 찾기 위해 테스트해본 것이고 마지막 로그를 확인하면 됩니다.
- 쿼리 로그에 대한 정보가 있는 것을 확인할 수 있습니다.
- 각 결과가 의미하는 것은 다음과 같습니다.

```sql
start_time: 쿼리가 시작된 시간
user_host: 쿼리를 실행한 사용자
query_time: 쿼리가 실행되는 데 걸린 시간
lock_time: 테이블 잠금에 대한 대기 시간
rows_sent: 실제 클라이언트에 전달된 레코드의 수
rows_examined: 쿼리가 처리되기 위해 실제 접근한 레코드의 수
db: 쿼리가 수행된 데이터베이스
sql_text: 실제 수행된 쿼리
```

### 서브쿼리를 조인문으로 바꿔 최적화 진행

- 그러면 서브쿼리를 CTE로 분리하여 조인문으로 바꿔 최적화를 해보겠습니다!

```sql
-- 최대 급여를 찾는 서브쿼리를 CTE(Common Table Expression)로 분리하여 최적화
WITH MaxSalary AS (
    SELECT MAX(salary) AS max_salary
    FROM salaries
    WHERE from_date > '2000-01-01'
)
SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
JOIN MaxSalary ms ON s.salary = ms.max_salary
WHERE e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC, s.salary DESC;
```

<img width="704" alt="image" src="https://github.com/user-attachments/assets/e58c7a4e-962b-44f3-a6f6-ae740c5469f7">

- 1.93초로 최적화된 것을 확인할 수 있습니다.

### 인덱스를 사용하여 최적화

- 그러면 이번에는 인덱스를 사용하여 최적화해보겠습니다.
- 인덱스 생성

```sql
CREATE INDEX idx_salaries_from_date ON salaries(from_date);
CREATE INDEX idx_employees_hire_date ON employees(hire_date);
CREATE INDEX idx_salaries_salary ON salaries(salary);
```

<img width="523" alt="image" src="https://github.com/user-attachments/assets/a96d050d-f39d-4826-8fd7-745163e513b0">

- 추가한 인덱스가 잘 적용되어있는 것을 확인할 수 있습니다.

<img width="683" alt="image" src="https://github.com/user-attachments/assets/233dc1d1-d3cb-4e37-89d2-551c0594129d">

- 쿼리에 걸리는 시간이 0.56초로 유의미하게 줄어든 것을 확인할 수 있습니다.

### 다음 테스트를 위한 인덱스 삭제

```sql
mysql> drop index idx_salaries_from_date on salaries;

mysql> drop index idx_employees_hire_date on employees;

mysql> drop index idx_salaries_salary on salaries;

mysql> show index from salaries;

mysql> show index from employees;
```

<img width="520" alt="image" src="https://github.com/user-attachments/assets/e033e364-b8cb-4025-8dd8-bd2aadeb5948">

## **2. 대량의 데이터를 검색하는 쿼리 최적화**

### 쿼리 실행

```sql
SELECT *
FROM employees
WHERE hire_date BETWEEN '1990-01-01' AND '2000-12-31'
AND emp_no IN (
    SELECT emp_no
    FROM salaries
    WHERE salary > 60000
)
ORDER BY hire_date DESC, emp_no ASC;
```

<img width="520" alt="image" src="https://github.com/user-attachments/assets/d7efb590-71ed-4518-845a-031e0dbd64db">

3.05초가 걸리는 것을 확인할 수 있습니다.

### 인덱스를 사용하여 최적화

- 인덱스 생성 후 확인

```sql
CREATE INDEX idx_employees_hire_date ON employees (hire_date);

CREATE INDEX idx_salaries_salary ON salaries (salary);

CREATE INDEX idx_employees_emp_no ON employees (emp_no);

CREATE INDEX idx_salaries_emp_no ON salaries (emp_no);

SHOW INDEX FROM employees;

SHOW INDEX FROM salaries;
```

<img width="705" alt="image" src="https://github.com/user-attachments/assets/be76fd16-9e79-4e91-baca-35b6d8acd612">

- 인덱스가 잘 추가된 것을 확인할 수 있습니다.
- 그러면 같은 쿼리문을 실행해보겠습니다.

```sql
SELECT *
FROM employees
WHERE hire_date BETWEEN '1990-01-01' AND '2000-12-31'
AND emp_no IN (
    SELECT emp_no
    FROM salaries
    WHERE salary > 60000
)
ORDER BY hire_date DESC, emp_no ASC;
```

<img width="524" alt="image" src="https://github.com/user-attachments/assets/5f7d5569-8ab5-4291-bc5b-bbb1cf32f92c">

- 1.02초로 쿼리 실행 시간이 단축된 것을 확인할 수 있습니다!
