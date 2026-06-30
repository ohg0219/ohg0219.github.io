---
title: "EXPLAIN은 빠른데 실측은 28초"
description: "커버링 인덱스로 잰 플랜이 거짓말한다 — 단일 인덱스와 비대해진 테이블의 함정"
date: 2026-06-30 09:50:45 +0900
categories: [backend, infra]
tags: [mariadb, query-optimization, covering-index, force-index, explain]
---

관리자 목록 화면 하나가 열릴 때마다 **30초씩** 멈췄습니다. 그런데 같은 화면이 6개 카테고리를 병렬로 긁어오는데, 나머지 5개는 0.3초면 끝납니다. 딱 한 카테고리만 100배 느렸습니다.

> 더 이상한 건, 그 느린 쿼리를 `EXPLAIN ANALYZE`로 떠보면 **164ms**가 나온다는 점이었습니다. 측정값은 멀쩡한데 운영은 30초. 이 글은 그 164ms가 어떻게 저를 한참 헤매게 했는지에 대한 기록입니다.

## 같은 화면, 100배 차이

목록 화면은 예금·적금 같은 상품 종류별로 따로 쿼리를 날려 메모리에서 합칩니다. 로그를 보면 차이가 한눈에 들어왔습니다.

| 카테고리 | 행수 | 조회 시간 |
|---|---|---|
| product:crawler (문제) | 25,392 | **34,828ms** |
| saving:crawler | 10,862 | 320ms |
| freeSaving:crawler | 11,733 | 324ms |

행수는 2배인데 시간은 100배. **"데이터가 많아서 느린 게 아니다"** 는 신호였습니다. 같은 구조, 같은 인덱스를 가진 테이블인데 한쪽만 플랜이 깨지고 있다는 뜻이죠.

## 164ms의 거짓말

가장 먼저 한 일은 당연히 `EXPLAIN ANALYZE`. 느린 쿼리의 골격만 떠봤습니다.

```sql
EXPLAIN ANALYZE
SELECT p.id
FROM code c
  JOIN store s ON s.category_cd = c.category_cd AND s.company_cd = c.company_cd
  JOIN product_crawler p ON p.store_id = s.id
WHERE c.use_yn = 'Y' AND s.use_yn = 'Y' AND p.use_yn = 'Y';
```

결과는 **164ms**. 마지막 노드도 깔끔했습니다.

```
-> Covering index lookup on p using product_crawler_store_id_use_yn_IDX
     (cost=1.06 rows=286) (actual time=0.00482..0.0053 rows=1.73 loops=14633)
```

플랜은 완벽한데 운영은 30초. 그래서 저는 **"옵티마이저가 통계에 따라 가끔 나쁜 조인 순서를 고른다(플랜 진동)"** 고 결론지었습니다. 처방도 그럴듯했습니다.

- `STRAIGHT_JOIN` 으로 조인 순서를 `code → store → member → product` 로 못 박고
- `NOT EXISTS` 상관 서브쿼리를 활성 회원 `LEFT JOIN ... IS NULL` anti-join 으로 치환

조인 순서를 강제했으니 이제 대용량 테이블이 드라이빙되는 30초 플랜은 다시 안 나올 거라 믿었습니다.

## 배포했는데 또 28초

```
deposit:crawler 데이터 조회 시작 / elapsed : 28092 ms / count : 25380
```

배포 후에도 그대로였습니다. 이 순간이 사실 가장 중요했습니다. **조인 순서를 강제했는데도 안 풀렸다는 건, 병목이 조인 순서가 아니었다는 뜻**이니까요. 진단 방향 자체가 틀린 겁니다.

여기서 처음 던졌어야 할 질문으로 돌아갔습니다. **"164ms가 나온 그 쿼리는, 정말 운영에서 도는 그 쿼리인가?"**

## 측정 쿼리가 실제 쿼리가 아니었다

아니었습니다. 제가 떠본 진단 쿼리는 `SELECT p.id` — **PK 한 컬럼**만 가져왔습니다. PK 는 인덱스에 이미 포함돼 있으니, MySQL 은 인덱스만 읽고 끝냅니다. 이게 **커버링 인덱스(covering index)** 입니다. 테이블 본문(클러스터드 인덱스)을 아예 건드리지 않죠. 그래서 164ms.

하지만 실제 매퍼 쿼리는 다릅니다.

```sql
SELECT p.id, p.product_nm, p.payment_term, p.term_6, p.term_12,
       p.term_24, p.term_36, p.post_date, p.mod_dt, ...
```

`product_nm`, `term_*`, `post_date` … **전부 인덱스에 없는 컬럼**입니다. 그러면 흐름이 완전히 달라집니다.

1. 인덱스로 조건에 맞는 PK 들을 찾고
2. 그 PK 하나하나로 **테이블 본문을 랜덤 액세스(lookup)** 해서 나머지 컬럼을 읽어옴

활성 상품이 약 2.5만 건이니, **테이블 lookup 을 2.5만 번** 하는 겁니다. 커버링 인덱스로 측정할 땐 이 2번 단계가 통째로 사라져 있었던 거죠. 측정값이 거짓말을 한 게 아니라, **저는 운영과 다른 쿼리를 측정하고 있었습니다.**

(중간에 "`ANALYZE TABLE` 치면 잠깐 빨라졌다 다시 느려진다"는 현상도 있었는데, 이건 통계 갱신 과정에서 페이지를 읽어 버퍼풀이 잠깐 워밍됐다가 evict 되면서 다시 디스크 random I/O 로 돌아가는 거였습니다. 같은 그림의 다른 증상이었습니다.)

## 진짜 범인

이번엔 **실제 컬럼을 전부 포함해서**, 그것도 느린 상태 그대로 `EXPLAIN ANALYZE` 를 떴습니다.

```
-> Nested loop inner join (actual time=10.9..28158 rows=25380 loops=1)
   ...앞단(code → store → member anti-join)은 103ms로 멀쩡...
   -> Index lookup on p using product_crawler_store_id_IDX (store_id=s.id)
        (actual time=0.0473..1.87 rows=581 loops=14631)   ← store당 581행을 읽음
   -> Filter: (p.use_yn = 'Y')  (rows=1.73)                ← 그중 1.73행만 생존
```

28초 전부가 마지막 `product_crawler` 접근에서 나왔습니다. 두 가지가 겹쳤습니다.

**(1) 테이블 비대화.** 테이블 실제 크기를 세어보니 충격적이었습니다.

```sql
SELECT COUNT(*) total, SUM(use_yn='Y') active FROM product_crawler;
-- total: 9,190,071 / active: 27,394  (0.3%)
```

크롤러가 매 회차 새 데이터를 INSERT 하면서, 이전 회차를 지우지 않고 `use_yn='N'` 으로만 두고 있었습니다. **919만 행 중 99.7%가 죽은 과거 데이터**. (saving 테이블도 495만 행이었지만, 절반 크기라 버퍼풀에 상주하며 간신히 버티고 있었습니다.)

**(2) 옵티마이저의 단일 인덱스 선택.** `(store_id, use_yn)` 복합 인덱스가 분명히 있는데도, 옵티마이저는 `store_id` **단일** 인덱스를 골랐습니다. 그래서 store 하나당 과거 회차까지 **581행**을 다 읽고, 테이블 lookup 까지 한 뒤에야 `use_yn='Y'` 로 걸러 1.73행만 남깁니다.

`14,631 loops × 581행 ≈ 850만 행` 을 읽어서 버리고 있었던 겁니다.

## 한 줄 고치고 115배

해결은 의외로 단출했습니다. 매퍼 쿼리에서 두 가지만 바꿨습니다.

```sql
-- before
JOIN product_crawler p ON p.store_id = s.id
WHERE ... AND p.use_yn = 'Y'

-- after
JOIN product_crawler p
     FORCE INDEX (product_crawler_store_id_use_yn_IDX)
     ON p.store_id = s.id AND p.use_yn = 'Y'
WHERE ...
```

- `(store_id, use_yn)` 복합 인덱스를 `FORCE INDEX` 로 **강제**하고
- `use_yn='Y'` 를 WHERE 에서 **조인 ON 절로 이동** (INNER JOIN 이라 결과는 불변)

이러면 `use_yn='Y'` 필터가 **인덱스 단계**에서 걸립니다. store 당 581행을 읽던 게 1.73행으로 줄죠.

```
-> Index lookup on p using product_crawler_store_id_use_yn_IDX (store_id=s.id, use_yn='Y')
     (actual time=0.00995..0.0105 rows=1.73 loops=14631)
전체: actual time=3.35..243 rows=25380
```

**28,158ms → 243ms. 약 115배.** 결과 행수 25,380 은 그대로라 정합성도 유지됐습니다.

## 가져갈 세 가지

- **측정 쿼리는 "운영에서 도는 그 쿼리" 여야 한다.** SELECT 컬럼 하나만 달라도 커버링 인덱스가 테이블 lookup 을 통째로 숨겨, 진단을 정반대로 끌고 갑니다. 골격만 떠서 빠르면 그건 골격이 빠른 거지 내 쿼리가 빠른 게 아닙니다.
- **`EXPLAIN` 의 rows 는 추정치다.** 의심되면 `EXPLAIN ANALYZE` 로, 그것도 **느린 상태에서** actual time/rows/loops 를 봐야 진짜 비용이 드러납니다. 빠른 상태에서 뜬 플랜은 "지금 빠른 이유"만 말해줄 뿐입니다.
- **인덱스 힌트는 증상 완화일 뿐이다.** 근본 원인은 919만 행 중 99.7%가 죽은 데이터라는 테이블 비대화였습니다. 과거 회차를 정리하는 배치가 없으면, 인덱스로 버틴 것도 결국 다시 느려지고 다른 테이블도 같은 임계점을 향해 갑니다. (그리고 애초에 50건만 필요한 화면이 56,459행을 메모리로 끌어와 정렬하는 구조라면, 그것도 따로 손봐야 할 근본이고요.)

덧붙이자면, 이 디버깅은 AI 와 페어로 진행했는데 — 처음 "플랜 진동" 오진과 STRAIGHT_JOIN 헛다리는 둘 다 그럴듯했고 둘 다 틀렸습니다. 방향을 되돌린 건 영리한 가설이 아니라 **"배포했는데 또 28초"라는 실측 한 줄**이었습니다. 결국 성능 문제 앞에서 믿을 건 추론이 아니라 실제로 느린 그 순간의 숫자더군요.
