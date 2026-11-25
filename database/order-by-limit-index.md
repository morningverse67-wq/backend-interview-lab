# DP-11. 최신순 목록 API가 느릴 때 인덱스는 어떻게 거나요?

## 30초 답변

거르는 컬럼 뒤에 정렬 컬럼을 붙인 복합 인덱스(`(user_id, created_at)`)를 걸어 정렬 단계를 없애겠습니다. 직접 재 보니 주문 20만 행에서 회원 한 명의 최근 20건을 가져올 때, `INDEX(user_id)`는 그 회원의 주문을 모두 읽어 임시 B-tree로 정렬했고(요청 1번에 VM 8,295단계, 1,000번에 192.8ms), `INDEX(user_id, created_at)`는 정렬 없이 20건만 읽고 멈췄습니다(133단계, 4.5ms).

## 조금 더 깊게

```sql
CREATE INDEX ix_orders_user_created ON orders(user_id, created_at);
SELECT id, created_at FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 20;
```

- 같은 `user_id` 안에서 이미 `created_at` 순서라, 뒤에서부터 LIMIT만큼 읽으면 끝납니다.
- 컬럼 순서는 같음 조건 컬럼 먼저, 정렬 컬럼 다음입니다.
- 실행 계획에서 `USE TEMP B-TREE FOR ORDER BY`(PostgreSQL은 `Sort` 노드)가 사라졌는지 확인합니다.

## 직접 측정한 근거

| 인덱스 | 정렬 단계(TEMP B-TREE) | 요청 1번 VM 단계 | 요청 1,000번 ms |
|---|---|---:|---:|
| INDEX(user_id) | 있음 | 8,295 | 192.8 |
| INDEX(user_id, created_at) | 없음 | 133 | 4.5 |

- 재현 코드: [backend-proof/036-order-by-limit-index](https://github.com/morningverse67-wq/backend-proof/tree/main/labs/036-order-by-limit-index)

<sub>Python 3.13.12, SQLite 3.50.4, macOS arm64. 실행 계획과 단계 수는 매번 같고 ms는 실행마다 달라집니다(192.8~249.6 대 4.5~6.6).</sub>

## 꼬리 질문

**Q. 시간 차이가 전부 정렬 때문인가요?**  
아닙니다. 이 쿼리는 `id, created_at`만 골라서 두 번째 인덱스가 커버링 인덱스도 됐습니다. 테이블을 다시 읽지 않는 효과가 섞여 있고, 정렬 단계가 사라진 것은 실행 계획으로 따로 확인했습니다.

**Q. 이 수치를 실제 서비스에 그대로 쓸 수 있나요?**  
SQLite 인메모리 실험이라 실제 DB 서버의 속도는 다를 수 있습니다. 정렬 단계가 사라지는 원리는 같습니다.

## 흔한 오답

- "`created_at`에 인덱스를 하나 더 걸면 됩니다" → 단일 컬럼 인덱스 두 개로는 `user_id`로 거른 결과의 정렬 단계가 그대로 남을 수 있습니다.

## 참고

- [SQLite: ORDER BY optimizations](https://www.sqlite.org/optoverview.html#order_by_optimizations)
- [PostgreSQL: Indexes and ORDER BY](https://www.postgresql.org/docs/current/indexes-ordering.html)
