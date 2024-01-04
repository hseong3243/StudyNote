```sql
select * from employees
where emp_no between 10001 and 10010
order by first_name
limit 0, 5;
```
위의 쿼리는 다음과 같은 순서로 실행된다.
1. `employees` 테이블에서 `where` 절의 검색 조건에 일치하는 레코드를 전부 읽어 온다.
2. 1번에서 읽어온 레코드를 `first_name` 컬럼값에 따라 정렬한다.
3. 정렬된 결과에서 상위 5건만 사용자에게 반환한다.

limit의 특성은 필요한 레코드 건수만 준비되면 즉시 쿼리를 종료한다. 

단, MySQL 서버가 limit에 해당하는 결과를 만들어내기 위해 어떠한 작업을 수행했는지 주의해야 한다.
```sql
select * from salaries order by salary limit n, m;
```
우의 쿼리에서 n, m이 굉장히 커지는 경우 쿼리 실행에 상당히 오랜 시간이 걸린다. `limit 1000000, 10` 인 경우 1000010건의 레코드를 읽은 후, 1000000건은 버리고 마지막 10건만 사용자에게 반환한다. 

limit 조건이 커질 가능성이 있는 경우 다음과 같이 `where` 절로 읽어야 할 위치를 찾고 그 위치에서 10개만 읽는 형태의 쿼리를 사용하는 것이 좋다.
```sql
select * from salaries
where salary >= 30000 and not (salary=30000 and emp_no<=270000)
order by salary limit 0, 10;
```
`not (salary=30000 and emp_no<=270000)`은 이전 페이지에서 이미 조회했던 건을 제외하기 위한 조건이다. 중복을 허용하는 인덱스인 경우 단순히 마지막 salary 값보다 큰거나 같은 값을 조회하는 경우 중복이나 누락이 발생할 수 있다. 중복이나 누락을 제외하기 위해서는 인덱스가 몇 개의 컬럼으로 구성돼 있는지, 유니크한지에 따라 달라질 수 있으니 주의하자.

