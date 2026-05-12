---
title: "MySQL only_full_group_by 에러 해결기"
date: 2026-05-12 16:30:00 +0900
categories: [backend]
tags: [mysql, sql, group-by, only-full-group-by, sql-mode, mybatis]
---

> **MariaDB 에선 잘 돌아가던 쿼리가, MySQL 8 로 옮긴 순간 터졌습니다.**
> 코드를 한 줄도 안 바꿨는데 말이죠.

운영 DB 를 MariaDB 에서 MySQL 8 로 옮긴 뒤, 한동안 멀쩡하게 돌아가던 모니터링 화면이 갑자기 에러를 뱉기 시작했습니다. 로그를 열어보니 처음 보는 이름의 예외가 떠 있었습니다.

```
java.sql.SQLSyntaxErrorException:
Expression #1 of SELECT list is not in GROUP BY clause and contains
nonaggregated column 'app_db.parent.parent_name' which is not functionally
dependent on columns in GROUP BY clause;
this is incompatible with sql_mode=only_full_group_by
```

쿼리는 분명 어제까지도 잘 돌던 SQL 입니다. 바뀐 건 DB 엔진뿐. 그런데 왜 지금에서야 터진 걸까요.

## 어디서 터진 건가

문제가 된 매퍼 쿼리는 대략 이런 모양이었습니다 (스키마·테이블·컬럼명은 모두 익명화).

```sql
SELECT  parent.parent_name AS parentName,
        parent.parent_label AS parentLabel,
        main_name           AS mainName,
        COUNT(1)            AS cnt
  FROM  app_db.main_tbl    AS main
  INNER JOIN app_db.parent_tbl AS parent
          ON parent.parent_key1 = main.parent_key1
         AND parent.parent_key2 = main.parent_key2
  LEFT  JOIN app_db.child_tbl  AS child
          ON child.main_id     = main.seq
         AND child.active_flag = 'Y'
 WHERE  0 = 0
   AND  main.parent_key1 = ?
   AND  main.active_flag = 'Y'
 GROUP BY main_name
HAVING cnt = 0
```

자세히 보면 이상한 게 보입니다.

- `SELECT` 절의 비집계 컬럼은 **세 개** — `parent.parent_name`, `parent.parent_label`, `main_name`
- 그런데 `GROUP BY` 에는 **한 개**만 — `main_name`

`main_name` 하나로 그룹을 묶었는데, 같은 `main_name` 안에 `parent_name` 값이 여러 개일 수도 있는 상황. MySQL 입장에서는 "그럼 어떤 값을 골라 보여줘야 하는데?" 라고 되묻는 것입니다. 그 대답이 바로 `only_full_group_by` 에러입니다.

MariaDB 는 이걸 그냥 묵인하고 "아무거나 하나 골라서" 결과를 내줬습니다. MySQL 8 은 묵인하지 않습니다.

## 왜 MariaDB 는 OK, MySQL 은 NG 인가

`sql_mode` 를 확인해 보면 이유가 명확해집니다.

```sql
SELECT VERSION();
SELECT @@SESSION.sql_mode;
SELECT @@GLOBAL.sql_mode;
```

운영 서버 결과:

```
mysql_version : 8.0.45-36

sql_mode (SESSION = GLOBAL):
  ONLY_FULL_GROUP_BY,
  STRICT_TRANS_TABLES,
  NO_ZERO_IN_DATE,
  NO_ZERO_DATE,
  ERROR_FOR_DIVISION_BY_ZERO,
  NO_ENGINE_SUBSTITUTION
```

여기서 가장 앞에 박혀 있는 `ONLY_FULL_GROUP_BY` — 이게 MySQL **5.7 이후 기본값**입니다. 표준 SQL 에 맞춰 "GROUP BY 에 없는 비집계 컬럼은 SELECT 에 쓰지 마라" 를 강제하는 모드죠.

반면 MariaDB 는 이 옵션이 기본적으로 빠져 있어, 같은 쿼리가 그냥 통과합니다. MariaDB → MySQL 마이그레이션을 했다면 거의 무조건 한 번은 만나는 함정입니다.

## 세 가지 길

해결 선택지는 크게 세 가지입니다.

**① 쿼리를 표준에 맞게 고친다**
SELECT 의 비집계 컬럼을 모두 GROUP BY 에 넣는다.

```sql
GROUP BY parent.parent_name, parent.parent_label, main_name
```

**② `ANY_VALUE()` 로 감싼다**
"이 컬럼은 그룹 내에서 아무 값이나 골라도 된다" 라고 명시한다.

```sql
SELECT ANY_VALUE(parent.parent_name) AS parentName,
       ANY_VALUE(parent.parent_label) AS parentLabel,
       main_name AS mainName,
       COUNT(1) AS cnt
  FROM ...
 GROUP BY main_name
```

**③ `sql_mode` 에서 `ONLY_FULL_GROUP_BY` 를 뺀다**

```sql
SET GLOBAL sql_mode = REPLACE(@@GLOBAL.sql_mode, 'ONLY_FULL_GROUP_BY,', '');
```

각각의 트레이드오프가 있습니다.

| 방법 | 장점 | 단점 |
|------|------|------|
| ① GROUP BY 보강 | 표준 준수, DB 이식성 ↑ | 동일 패턴 쿼리를 다 손봐야 함 |
| ② `ANY_VALUE()` | 의도가 코드에 명시됨 | MySQL 전용 함수 |
| ③ sql_mode 풀기 | 코드 무수정 | 다른 곳에서 또 모래 위에 집을 짓게 됨 |

## 우리가 택한 길

저는 **①번 — 쿼리 수정**을 골랐습니다.

이유는 두 가지.

첫째, `sql_mode` 를 푸는 건 "지금 당장의 에러"는 없애지만, **표준에서 어긋난 쿼리를 계속 양산할 수 있는 환경**을 만듭니다. 다음 개발자가 같은 함정에 또 빠지죠. 운영 DB 의 안전장치를 끄는 것 = 미래의 비용을 키우는 것.

둘째, 같은 매퍼 XML 안에 **동일 패턴의 쿼리가 3개** 더 있었습니다 (`selectListFreeSavingCrawler` / `selectListDepositCrawler` / `selectListSavingCrawler`). 한 곳만 고치고 넘어가면 나머지 두 개가 또 터질 게 뻔했죠. 어차피 일괄 수정이 필요한 상황.

수정은 단순합니다.

```diff
- GROUP BY main_name
+ GROUP BY parent.parent_name, parent.parent_label, main_name
```

세 쿼리 모두에 같은 패턴으로 적용. 커밋 메시지에는 원인까지 명시했습니다.

```
[BE] CrawlerMonitor SQL only_full_group_by 호환 처리

GROUP BY 에 SELECT 절 비집계 컬럼을 추가하여
sql_mode=only_full_group_by 환경에서 발생하던
SQLSyntaxErrorException 해결.
```

## 가져갈 한 줄

MariaDB → MySQL 전환에서 가장 먼저 확인해야 할 것은 **`sql_mode`** 입니다. 특히 `ONLY_FULL_GROUP_BY`. 코드를 한 줄도 안 바꿨는데 쿼리가 터진다면, 99% 는 DB 엔진이 들고 있는 안전장치가 켜진 쪽이지 내 쿼리가 갑자기 망가진 게 아닙니다.

그리고 그 안전장치를 끄지 마세요. 안전장치는 이유가 있어서 켜져 있습니다.

## 참고

- [MySQL 8.0 Reference Manual — MySQL Handling of GROUP BY](https://dev.mysql.com/doc/refman/8.0/en/group-by-handling.html)
- [MySQL 8.0 Reference Manual — `sql_mode`](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sqlmode_only_full_group_by)
