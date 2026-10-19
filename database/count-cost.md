# DB-08. COUNT(*)는 왜 느려지고, 대안은 무엇인가요?

## 30초 답변

`COUNT(*)`는 조건에 맞는 행이나 인덱스 항목을 하나씩 지나가야 하므로, 데이터가 늘면 같이 느려집니다. 아래 실험에서 100만 건 중 59.9만 건을 세는 데 인덱스 없이 21.16ms, `status` 인덱스로 6.12ms가 걸렸습니다.
자주 보여 줘야 하는 개수라면 카운터 테이블에 미리 세 두면 한 행만 읽어서 0.0005ms 수준이 되지만, 쓰기마다 카운터를 고쳐야 해서 INSERT가 2.90배, DELETE가 4.73배 느려졌습니다.
그리고 "있는지"만 알면 되는 곳에서는 `COUNT(*) > 0` 대신 `EXISTS`를 써서 첫 번째 행에서 멈추게 합니다.

## 조금 더 깊게

```sql
-- 1) 인덱스로 세는 범위를 줄인다 (테이블 대신 인덱스만 읽음)
CREATE INDEX idx_orders_status ON orders(status);
SELECT COUNT(*) FROM orders WHERE status = 'PAID';

-- 2) 미리 세 둔다 (쓰기 때 같이 갱신)
CREATE TABLE order_counts (status TEXT PRIMARY KEY, cnt INTEGER NOT NULL);
-- INSERT/UPDATE/DELETE 트리거 또는 애플리케이션에서 cnt를 +1/-1

-- 3) 개수가 아니라 존재 여부만 필요하면
SELECT EXISTS (SELECT 1 FROM orders WHERE user_id = ? AND status = 'CANCELLED');
```

- **인덱스**: 조건 컬럼에 인덱스가 있으면 테이블 행 대신 더 작은 인덱스 항목만 셉니다(실행 계획 `USING COVERING INDEX`). 그래도 맞는 항목 수만큼은 지나가므로 결과가 많으면 여전히 비쌉니다.
- **카운터 테이블**: 읽기는 한 행이라 데이터 크기와 상관없지만, 미리 정한 기준(상태별 등)만 셀 수 있고 쓰기 비용이 늘어납니다. 여러 트랜잭션이 같은 카운터 행을 고치면 그 행에서 서로 기다립니다. MySQL 문서도 카운터 테이블을 권하면서 동시 갱신이 많으면 잘 확장되지 않을 수 있다고 적고 있습니다.
- **정확하지 않아도 되는 경우**: "약 120만 건"처럼 보여 줘도 되면 MySQL `SHOW TABLE STATUS`의 추정치나 통계 테이블을 씁니다. 페이지 목록이라면 전체 개수 대신 `LIMIT n + 1`로 다음 페이지가 있는지만 확인하는 방법도 있습니다.

## 직접 측정한 근거

![주문 100만 건에서 status = 'PAID' COUNT(*)가 인덱스 없이 21.16ms, status 인덱스 6.12ms, 카운터 테이블 0.0005ms. 대신 카운터 트리거 때문에 INSERT 2만 건 12.02ms에서 34.87ms(2.90배), UPDATE 3.52배, DELETE 4.73배 느려짐. EXISTS는 금방 찾으면 18.10ms에서 0.0005ms, 하나도 없으면 1.0배인 비교 그래프](../assets/db-08-count-cost.png)

주문 100만 건(PAID 60% / SHIPPED 30% / CANCELLED 10%), `user_id` 인덱스 1개, SQLite 메모리 DB. 카운터 테이블은 INSERT / UPDATE / DELETE 트리거로 갱신하고, 모든 방법의 결과가 같은지와 쓰기 뒤 카운터가 `COUNT(*) ... GROUP BY status`와 같은지 확인. 시간은 쿼리 1회당 값(10회 평균, 3회 중 최소값).

| `COUNT(*) WHERE status = 'PAID'` (599,890건) | 실행 계획 | 시간 (ms) |
|---|---|---:|
| 인덱스 없음 | `SCAN orders` | 21.1568 |
| `status` 인덱스 | `SEARCH ... USING COVERING INDEX idx_orders_status` | 6.1215 |
| 카운터 테이블 읽기 | 기본 키로 한 행 | 0.0005 |

| "하나라도 있나?" (인덱스 없는 `amount`) | `COUNT(*) > 0` (ms) | `EXISTS` (ms) |
|---|---:|---:|
| 금방 찾음 (`amount >= 50000`) | 18.1045 | 0.0005 |
| 하나도 없음 (`amount >= 1000000`) | 15.4496 | 15.4158 |

| 카운터 트리거의 쓰기 비용 (10만 건에서 시작) | 트리거 없음 (ms) | 카운터 트리거 (ms) | 느려진 배율 |
|---|---:|---:|---:|
| INSERT 2만 건 | 12.02 | 34.87 | 2.90x |
| status UPDATE 1만 건 | 1.20 | 4.21 | 3.52x |
| DELETE 1만 건 | 2.59 | 12.28 | 4.73x |

- 인덱스는 21.16ms → 6.12ms(3.5배)로 줄였지만 59.9만 개 항목을 세는 일은 남아 있습니다. 카운터는 읽기를 거의 없앤 대신 그 비용을 모든 쓰기에 나눠 냅니다.
- `EXISTS`가 빠른 건 맞는 행을 일찍 만났을 때뿐입니다. 맞는 행이 없으면 끝까지 읽어서 15.45ms vs 15.42ms로 같았습니다.
- 재현 코드: [backend-lab/009-count-cost](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/009-count-cost)

<sub>Python 3.13 / SQLite 3.50, in-memory DB, 10회 평균의 3회 중 최소값. 0.0005ms는 쿼리 한 번을 호출하는 비용과 비슷한 수준이라 배율로 비교하지 않았습니다. 환경에 따라 수치는 달라질 수 있음.</sub>

## 꼬리 질문

**Q. `COUNT(*)`, `COUNT(1)`, `COUNT(컬럼)`은 뭐가 다른가요?**  
`COUNT(*)`와 `COUNT(1)`은 행 수를 세고, MySQL InnoDB 문서에 따르면 둘은 같은 방식으로 처리돼 성능 차이가 없습니다. `COUNT(컬럼)`은 그 컬럼이 NULL이 아닌 행만 세므로 결과 자체가 다를 수 있습니다.

**Q. 조건 없는 전체 개수도 매번 느린가요?**  
DB마다 다릅니다. 이 실험의 SQLite는 가장 작은 인덱스를 훑어서 세서 100만 건에 0.0573ms였고, 1만 건에서는 카운터와 차이가 0.0002ms라 카운터를 둘 이유가 없었습니다. MySQL InnoDB는 트랜잭션마다 보이는 행 수가 달라 테이블 전체 행 수를 따로 저장하지 않으므로, 가장 작은 보조 인덱스를 끝까지 훑어서 셉니다(이 실험에서 MySQL은 재지 않음).

**Q. 카운터 테이블이 실제 개수와 어긋나면 어떻게 하나요?**  
카운터 갱신이 원래 쓰기와 같은 트랜잭션에 있어야 어긋나지 않습니다. 이 실험은 트리거로 같은 트랜잭션 안에서 갱신했고, 쓰기 뒤 `COUNT(*) GROUP BY status`와 값이 같았습니다. 애플리케이션이나 캐시에서 따로 갱신한다면 주기적으로 `COUNT(*)`로 다시 맞추는 작업을 둡니다.

## 흔한 오답

- "인덱스를 걸면 `COUNT(*)`가 해결됩니다" → 읽는 양이 줄 뿐, 맞는 항목 수만큼은 셉니다. 이 실험에서도 3.5배였고, 데이터가 늘면 다시 느려집니다.
- "`EXISTS`가 `COUNT(*)`보다 항상 빠릅니다" → 맞는 행이 없으면 둘 다 끝까지 읽어서 1.0배였습니다. 이때는 조건 컬럼의 인덱스가 필요합니다.

## 참고

- [MySQL 8.4: Aggregate Function Descriptions (COUNT)](https://dev.mysql.com/doc/refman/8.4/en/aggregate-functions.html)
- [MySQL 8.4: InnoDB Restrictions and Limitations](https://dev.mysql.com/doc/refman/8.4/en/innodb-restrictions-limitations.html)
- [SQLite: CREATE TRIGGER](https://www.sqlite.org/lang_createtrigger.html)
- [SQLite: The EXISTS operator](https://www.sqlite.org/lang_expr.html#the_exists_operator)
