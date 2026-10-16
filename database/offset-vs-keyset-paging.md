# DB-04. OFFSET 페이징은 왜 느려지고, 커서 페이징은 언제 쓰나요?

## 30초 답변

`LIMIT 20 OFFSET 199980`은 DB가 앞의 199,980건을 실제로 읽은 다음 버리기 때문에, 페이지가 깊어질수록 읽는 양이 늘어 느려집니다.
커서(keyset) 페이징은 직전 페이지 마지막 행의 정렬 키를 기억했다가 `WHERE (created_at, id) < (?, ?)`처럼 그다음부터 읽어서, 인덱스로 시작 위치를 바로 찾고 깊이와 상관없이 일정한 속도가 나옵니다.
아래 실험에서는 1페이지를 본 뒤 새 글이 1건 들어와도 커서 방식의 2페이지에 중복이 0건이었습니다(OFFSET은 1건 중복).
대신 "N페이지로 바로 이동"은 못 하고, 정렬 키가 유일해야 해서 무한 스크롤이나 배치 처리에 잘 맞고, 페이지 번호 UI에는 OFFSET이 여전히 쓰입니다.

## 조금 더 깊게

```sql
-- OFFSET
SELECT id, created_at, title FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20 OFFSET ?;

-- 커서(keyset): 클라이언트가 직전 페이지 마지막 행의 (created_at, id)를 보낸다
SELECT id, created_at, title FROM posts
WHERE (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

- 인덱스 `(created_at, id)`가 있어도 OFFSET 쿼리의 실행 계획은 `SCAN ... USING INDEX`입니다. 인덱스 순서대로 처음부터 읽으면서 건너뛸 행을 하나씩 세고 버립니다.
- 커서 쿼리는 `SEARCH ... USING INDEX (created_at<?)`로 시작 위치를 바로 찾고 20건만 읽습니다.
- 커서에는 **유일한 정렬 키**가 필요합니다. `created_at`처럼 중복될 수 있는 컬럼만 쓰면 페이지 경계에서 같은 값의 나머지 행을 건너뜁니다. 보통 `id`를 뒤에 붙이고, 인덱스도 같은 순서로 만듭니다.
- 페이지 번호 대신 "다음 페이지" 토큰을 주고받기 때문에 API 응답에 `nextCursor` 같은 값을 넣는 형태가 됩니다.

## 직접 측정한 근거

![1페이지는 OFFSET과 커서가 0.67ms로 같고, 10,000페이지는 OFFSET 187.96ms 대 커서 0.77ms로 244.7배 차이. 10,000페이지로 바로 이동할 때는 OFFSET이 1.90ms로 커서 99.17ms보다 52.3배 빠른 비교 그래프](../assets/db-04-offset-vs-keyset-paging.png)

게시글 30만 건, `created_at`은 분 단위라 같은 시각에 평균 7건 정도가 겹침. SQLite in-memory, 인덱스 `(created_at, id)`, 한 페이지 20건. 각 페이지를 100번 조회한 합계.

| 페이지 | 건너뛰는 행 | OFFSET (ms) | 커서 (ms) | 배율 |
|---:|---:|---:|---:|---:|
| 1 | 0 | 0.67 | 0.67 | 1.0x |
| 100 | 1,980 | 2.50 | 0.75 | 3.3x |
| 1,000 | 19,980 | 18.94 | 0.75 | 25.3x |
| 10,000 | 199,980 | 187.96 | 0.77 | 244.7x |

| 상황 | OFFSET | 커서 |
|---|---|---|
| 1페이지를 본 뒤 새 글 1건, 2페이지 | 1페이지 행 1건이 중복 | 중복 0건 |
| 30만 건 전체를 끝까지 넘기기 | - | `(created_at, id)`: 누락 0건 / `created_at`만: 44,509건 누락 |
| 커서 없이 10,000페이지로 바로 이동 | 1.90ms | 10,000페이지를 넘겨서 99.17ms (52.3배 느림) |

- OFFSET 시간은 건너뛰는 행 수에 거의 비례해서 늘었고, 커서는 1페이지든 10,000페이지든 0.7ms대였습니다. 1페이지에서는 차이가 없었으니, 앞쪽 몇 페이지만 보는 화면이면 OFFSET도 충분합니다.
- 반대로 특정 페이지로 바로 가야 하면 커서는 앞 페이지를 전부 넘겨야 해서 OFFSET이 52.3배 빨랐습니다.
- 재현 코드: [backend-lab/005-offset-vs-keyset-paging](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/005-offset-vs-keyset-paging)

<sub>Python 3.13 / SQLite 3.50, 3회 중 최소값. 환경에 따라 수치는 달라질 수 있음.</sub>

## 꼬리 질문

**Q. 커서 페이징에서 정렬 컬럼에 중복 값이 있으면 어떻게 하나요?**  
유일한 컬럼(보통 PK)을 정렬과 커서에 함께 넣습니다. `WHERE (created_at, id) < (?, ?)`처럼 비교하고, 행 값 비교를 지원하지 않는 DB에서는 `created_at < ? OR (created_at = ? AND id < ?)`로 풀어 씁니다. 위 실험에서 `created_at`만 쓰면 30만 건 중 44,509건이 빠졌습니다.

**Q. 페이지 번호 UI가 꼭 필요하면 OFFSET의 느림은 어떻게 줄이나요?**  
이동 가능한 페이지 수를 제한하거나(예: 앞쪽 100페이지까지), 인덱스만으로 PK를 먼저 찾은 뒤 본문 컬럼을 조인하는 방식(지연 조인)으로 건너뛰는 비용을 줄입니다. 그래도 건너뛰는 행을 읽는 구조 자체는 남습니다.

**Q. 커서 값은 클라이언트에 그대로 노출해도 되나요?**  
정렬 키 값을 그대로 보내도 동작은 하지만, 보통 Base64 등으로 묶어 불투명한 토큰으로 주고받습니다. 정렬 기준이 바뀌어도 API 형태를 유지하기 쉽고, 클라이언트가 값을 임의로 조작하는 것도 줄일 수 있습니다.

## 흔한 오답

- "OFFSET도 인덱스를 타니까 깊은 페이지도 빠릅니다" → 인덱스를 따라가도 건너뛸 행을 하나씩 읽습니다. 위 측정에서 10,000페이지는 1페이지보다 약 280배 느렸습니다(0.67ms → 187.96ms).
- "커서 페이징이 항상 더 좋습니다" → 1페이지에서는 차이가 없었고, 특정 페이지로 바로 이동할 때는 OFFSET이 52.3배 빨랐습니다. 화면 요구사항에 따라 고릅니다.

## 참고

- [PostgreSQL: LIMIT and OFFSET](https://www.postgresql.org/docs/current/queries-limit.html)
- [MySQL 8.0: LIMIT Query Optimization](https://dev.mysql.com/doc/refman/8.0/en/limit-optimization.html)
- [SQLite: Row Values (Scrolling Window Queries)](https://www.sqlite.org/rowvalue.html)
- [SQLite: EXPLAIN QUERY PLAN](https://www.sqlite.org/eqp.html)
- 블로그 글: [OFFSET 페이징은 왜 뒤로 갈수록 느려질까](https://velog.io/@morningverse/OFFSET-페이징은-왜-뒤로-갈수록-느려질까)
