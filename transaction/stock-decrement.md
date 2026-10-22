# TX-03. 재고 차감 동시성 문제를 어떻게 해결하나요?

## 30초 답변

재고를 읽어서 앱에서 `qty > 0`을 검사한 뒤 쓰면, 여러 요청이 같은 "남은 1개"를 보고 모두 팔 수 있습니다. 가장 먼저 쓰는 방법은 검사와 차감을 한 문장으로 합친 원자적 UPDATE, `UPDATE items SET qty = qty - 1 WHERE id = ? AND qty > 0`이고, 바뀐 행이 0이면 품절로 처리합니다.
아래 실험에서 재고 100개에 주문 400건을 동시에 보내자 읽고-쓰기는 400개를 팔았고, 원자적 UPDATE는 정확히 100개만 팔았습니다. 재고가 충분한 주문 800건은 락을 쓴 방법보다 2.17배 빨랐습니다(634.9ms vs 1,379.7ms). 다만 여러 상품을 함께 차감하거나 다른 테이블을 봐야 하는 규칙이 있으면 트랜잭션이나 락이 필요합니다.

## 조금 더 깊게

```sql
-- 위험: 읽은 값으로 검사하고 계산한 값을 씀
SELECT qty FROM items WHERE id = 1;            -- 1을 읽음 (다른 요청도 1을 읽음)
UPDATE items SET qty = 0 WHERE id = 1;

-- 원자적 UPDATE: 검사와 차감이 한 문장, 바뀐 행 수가 곧 판매 성공 여부
UPDATE items SET qty = qty - 1 WHERE id = 1 AND qty > 0;

-- 여러 상품을 함께 차감: 하나라도 0행이면 전부 되돌림
BEGIN;
UPDATE items SET qty = qty - 1 WHERE id = 1 AND qty > 0;
UPDATE items SET qty = qty - 1 WHERE id = 2 AND qty > 0;   -- 0행이면 ROLLBACK
COMMIT;
```

- **차감만 `qty - 1`로 바꾸면 절반만 고쳐집니다**: 숫자는 맞게 줄지만 검사는 여전히 옛 값으로 합니다. 실험에서 107개가 팔리고 재고가 -7이 됐습니다.
- **원자적 UPDATE는 앱 작업을 락 밖에서 할 수 있습니다**: 낙관적 락과 비관적 락은 읽기와 쓰기 사이의 작업 시간 동안 충돌하거나 줄을 섭니다. 원자적 UPDATE는 DB가 잠깐 행을 잠그는 시간만 순서를 기다립니다.
- **MySQL InnoDB에서 왜 안전한가**: 일반 `SELECT`는 스냅샷을 읽지만, `UPDATE` 같은 잠금 문장은 가장 최근에 커밋된 값을 읽고 행을 잠근 뒤 조건을 봅니다. 그래서 `qty > 0` 검사가 다른 요청이 뺀 결과를 보고 이뤄집니다(MySQL에서는 직접 재지 않았습니다).
- **바뀐 행 수를 꼭 확인합니다**: JDBC `executeUpdate()`, JPA `@Modifying` 쿼리의 반환값이 0이면 품절입니다. 이걸 무시하면 재고는 안 줄었는데 주문은 성공하는 반대 문제가 생깁니다.

## 직접 측정한 근거

![재고 100개에 주문 400건을 보내면 읽고-쓰기는 400개를 팔고 재고 50개가 남았고, 앱에서 검사하고 qty - 1로 차감하면 107개를 팔고 재고가 -7이 됐다. 원자적 UPDATE, 낙관적 락, 행 락은 정확히 100개. 재고가 충분한 주문 800건은 원자적 UPDATE 634.9ms, 낙관적 락 1,375.6ms, 행 락 1,379.7ms(2.17배). 장바구니 주문은 UPDATE를 하나씩 커밋하면 A 재고 70개가 주문 없이 사라진 비교 그래프](../assets/tx-03-stock-decrement.png)

SQLite 파일 DB(WAL), 8스레드가 같은 상품을 1개씩 주문. 주문마다 1ms 앱 작업(주문 검증 흉내)을 하고, 읽고-쓰기 방식은 `SELECT`와 `UPDATE` 사이에, 원자적 UPDATE는 `UPDATE` 전에 락 밖에서 합니다. SQLite에는 `FOR UPDATE`가 없어서 행 락은 행마다 락을 두고 먼저 온 순서대로 넘겨주는 방식으로 흉내 냈습니다. 3회 실행의 지표별 중앙값입니다.

| 방법 | 재고 100 · 주문 400: 팔린 수량 | 남은 재고 | 재고 800 · 주문 800: 전체 시간 (ms) | 재시도 |
|---|---:|---:|---:|---:|
| 읽고-쓰기 (`SET qty = ?`) | 400 | 50 | 645.4 (재고 700 남음) | - |
| 읽고 검사 + `qty - 1` | 107 | -7 | 587.6 | - |
| 원자적 UPDATE | 100 | 0 | 634.9 | - |
| 낙관적 락 | 100 | 0 | 1,375.6 | 2,927 |
| 행 락(비관적 락) | 100 | 0 | 1,379.7 | - |

장바구니 주문 200건 (A 재고 100, B 재고 30, 주문마다 둘 다 1개씩): 원자적 UPDATE를 하나씩 커밋하면 30건이 팔리고 A 재고는 0, 즉 70개가 주문 없이 사라졌습니다. 같은 UPDATE를 트랜잭션으로 묶고 실패 시 `ROLLBACK`하면 A가 70개 남았습니다.

- 원자적 UPDATE, 낙관적 락, 행 락은 매번 정확히 재고만큼만 팔았고, 그중 원자적 UPDATE가 재시도 없이 가장 빨랐습니다. 품절이 빨리 나는 경우(재고 100 · 주문 400)에는 178.4ms vs 180.2ms로 시간 차이가 없었습니다. 행 락도 품절 뒤에는 바로 실패하기 때문입니다.
- 재고 800 · 주문 800에서 틀린 방법들도 비슷한 시간이 걸린 것은 SQLite가 쓰기를 파일 단위로 하나씩만 받기 때문으로 보이며, 다른 DB에서는 다를 수 있습니다.
- 재현 코드: [backend-lab/012-stock-decrement-concurrency](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/012-stock-decrement-concurrency)

<sub>Python 3.13 / SQLite 3.50, 파일 DB(WAL, synchronous=NORMAL), 3회 실행의 지표별 중앙값. 스레드 실행 순서에 따라 매번 조금씩 달라지고, 환경에 따라 수치는 달라질 수 있음.</sub>

## 꼬리 질문

**Q. 원자적 UPDATE로 부족한 경우는 언제인가요?**  
여러 행이 함께 바뀌어야 할 때와, 규칙이 그 행의 `WHERE`에 안 들어갈 때입니다. 장바구니처럼 상품 여러 개를 함께 빼야 하면 트랜잭션으로 묶어야 하고(실험에서 묶지 않으면 A 재고 70개가 사라짐), 1인당 구매 제한처럼 주문 테이블을 함께 봐야 하는 규칙은 비관적 락이나 낙관적 락으로 읽은 값을 지켜야 합니다. 여러 행을 잠글 때는 순서를 정해 두지 않으면 데드락이 생길 수 있습니다.

**Q. 재고가 음수가 되는 걸 DB에서도 막을 수 있나요?**  
`CHECK (qty >= 0)` 제약을 걸면 실수로 조건 없는 차감이 나가도 DB가 거절합니다. MySQL은 8.0.16부터 `CHECK` 제약을 실제로 검사합니다. 다만 거절은 에러로 오기 때문에, 품절 처리는 여전히 원자적 UPDATE의 바뀐 행 수로 하는 편이 깔끔합니다.

**Q. 이벤트처럼 한 상품에 주문이 몰리면 DB 행 하나가 병목이 되지 않나요?**  
됩니다. 원자적 UPDATE도 같은 행에 대한 쓰기는 한 번에 하나씩이라, 처리량의 상한은 그 행의 쓰기 속도입니다. 이럴 때 Redis `DECR` 같은 메모리 카운터로 먼저 걸러 내거나, 주문을 큐에 넣어 순서대로 처리하는 방법을 씁니다. 이 경우 DB와 카운터 값이 어긋났을 때 어떻게 맞출지도 함께 정해야 합니다(이 부분은 직접 재지 않았습니다).

## 흔한 오답

- "`@Transactional`을 붙이면 재고가 맞습니다" → 트랜잭션은 묶은 작업을 함께 커밋하거나 되돌릴 뿐, 다른 요청이 같은 값을 읽는 걸 막지 않습니다. MySQL 기본 격리 수준에서 일반 `SELECT`는 스냅샷을 읽어서 두 트랜잭션이 같은 재고를 볼 수 있습니다.
- "`qty = qty - 1`로 바꾸면 해결됩니다" → 검사를 앱에서 옛 값으로 하면 여전히 재고보다 많이 팝니다. 실험에서 107개를 팔고 재고가 -7이 됐습니다. `AND qty > 0`으로 검사도 같은 문장에 넣어야 합니다.

## 참고

- [MySQL 8.4: Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)
- [MySQL 8.4: Transaction Isolation Levels](https://dev.mysql.com/doc/refman/8.4/en/innodb-transaction-isolation-levels.html)
- [MySQL 8.4: UPDATE Statement](https://dev.mysql.com/doc/refman/8.4/en/update.html)
- [MySQL 8.4: CHECK Constraints](https://dev.mysql.com/doc/refman/8.4/en/create-table-check-constraints.html)
- [SQLite: UPDATE](https://www.sqlite.org/lang_update.html)
