# TX-02. 낙관적 락과 비관적 락은 언제 각각 쓰나요?

## 30초 답변

충돌이 얼마나 자주 나는지, 그리고 충돌했을 때 재시도와 실패를 감당할 수 있는지로 고릅니다. 낙관적 락은 락 없이 진행하고 쓸 때 `version`을 검사해서, 충돌이 드문 곳에서는 거의 비용이 없습니다. 비관적 락은 `SELECT ... FOR UPDATE`처럼 읽기 전에 행을 잠가서 다른 요청을 기다리게 합니다.
아래 실험에서 한 행에 `+1` 요청 1,600번을 보냈더니, 낙관적 락은 5,624번 재시도했고 느린 쪽 1% 요청(p99)이 174.32ms 걸렸습니다. 행을 먼저 잠그자 재시도는 0번, p99는 18.92ms였습니다. 다만 전체 처리 시간은 2,767.2ms와 2,771.7ms로 같았으니, 비관적 락은 처리량을 늘리는 도구가 아니라 지연을 고르게 하고 재시도 실패를 없애는 도구입니다.

## 조금 더 깊게

```sql
-- 낙관적 락: 쓰는 순간 버전 검사, 바뀐 행이 0이면 충돌
UPDATE items SET qty = 11, version = version + 1 WHERE id = 1 AND version = 3;

-- 비관적 락: 읽을 때 잠그고, COMMIT까지 같은 행을 잠그려는 요청은 대기
BEGIN;
SELECT qty FROM items WHERE id = 1 FOR UPDATE;
UPDATE items SET qty = 11 WHERE id = 1;
COMMIT;
```

- **처리량은 같고, 지연 모양이 다릅니다**: 한 행에 대한 쓰기는 어느 방법이든 한 번에 하나씩입니다. 낙관적 락은 운 좋은 요청은 바로 끝나지만(p50 1.66ms) 계속 충돌하는 요청이 생기고, 비관적 락은 모두 줄을 서서(p50 13.62ms) 대신 오래 밀리는 요청이 없습니다(최대 25.26ms).
- **락 범위가 좁아야 합니다**: 비관적 락은 잡은 범위만큼 줄을 세웁니다. 실험에서 1,000행에 흩어진 요청도 락 하나로 묶으면 1,080.9ms에서 2,692.4ms(2.49배)가 됐습니다. MySQL InnoDB는 `FOR UPDATE` 조건에 맞는 인덱스가 없으면 훑은 행을 모두 잠근다고 문서에 나와 있으니, 잠글 행은 인덱스로 찾아야 합니다.
- **락을 잡는 시간도 짧아야 합니다**: 줄을 선 요청은 앞 요청이 락을 잡고 있는 시간만큼 기다립니다. 락을 잡은 채 외부 API를 부르면 그 시간이 뒤에 선 모든 요청에 더해집니다.
- **JPA에서는**: 낙관적 락은 `@Version`, 비관적 락은 `@Lock(LockModeType.PESSIMISTIC_WRITE)`로 씁니다. MySQL에서 비관적 쓰기 락은 보통 `SELECT ... FOR UPDATE`로 나갑니다.

## 직접 측정한 근거

![한 행에 +1 요청 1,600번을 보내면 낙관적 락은 재시도 5,624번에 p99 174.32ms, 행을 먼저 잠그는 비관적 락은 재시도 0번에 p99 18.92ms. 전체 시간은 2,767.2ms와 2,771.7ms로 거의 같고, 1,000행에 흩어진 요청을 락 하나로 묶으면 1,080.9ms에서 2,692.4ms(2.49배)인 비교 그래프](../assets/tx-02-pessimistic-lock.png)

SQLite 파일 DB(WAL), 8스레드 x 200번 = `+1` 요청 1,600번. 요청마다 `SELECT` → 1ms 대기(애플리케이션 처리 시간 흉내) → `UPDATE`. SQLite에는 `FOR UPDATE`가 없어서 행 락은 행마다 락을 두고 먼저 온 순서대로 넘겨주는 방식으로 흉내 냈습니다. 지연은 요청 하나가 끝나기까지(대기, 재시도 포함) 걸린 시간이고, 3회 실행의 지표별 중앙값입니다. 모든 방법에서 매번 최종 값 1,600을 확인했습니다.

| 경우 | 방법 | 재시도 | 전체 시간 (ms) | p50 (ms) | p99 (ms) | 최대 (ms) |
|---|---|---:|---:|---:|---:|---:|
| 한 행에 몰림 | 낙관적 락 | 5,624 | 2,767.2 | 1.66 | 174.32 | 424.06 |
| 한 행에 몰림 | 행 락(비관적 락) | 0 | 2,771.7 | 13.62 | 18.92 | 25.26 |
| 1,000행에 흩어짐 | 낙관적 락 | 8 | 1,173.1 | 1.73 | 47.72 | 794.51 |
| 1,000행에 흩어짐 | 행 락(비관적 락) | 0 | 1,080.9 | 1.74 | 49.44 | 361.42 |
| 1,000행에 흩어짐 | 락 하나로 전부 | 0 | 2,692.4 | 13.44 | 14.46 | 16.29 |

- 한 행에 몰리면 비관적 락은 재시도를 없애고 p99를 174.32ms에서 18.92ms로 낮췄지만, 전체 시간은 줄지 않았고 중앙값은 1.66ms에서 13.62ms로 늘었습니다.
- 흩어진 경우는 낙관적 락도 재시도 8번으로 충분했습니다. 흩어진 경우의 p99가 40ms대인 것은 SQLite가 쓰기를 파일 단위로 하나씩만 받기 때문으로 보이며, 다른 DB에서는 다를 수 있습니다.
- 재현 코드: [backend-lab/011-pessimistic-lock](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/011-pessimistic-lock)

<sub>Python 3.13 / SQLite 3.50, 파일 DB(WAL, synchronous=NORMAL), 3회 실행의 지표별 중앙값. 스레드 실행 순서에 따라 매번 조금씩 달라지고, 환경에 따라 수치는 달라질 수 있음.</sub>

## 꼬리 질문

**Q. 비관적 락을 걸면 다른 요청의 일반 SELECT도 막히나요?**  
보통은 아닙니다. MySQL InnoDB의 기본 격리 수준에서 일반 `SELECT`는 잠금 없이 스냅샷을 읽으므로 기다리지 않고 커밋 전 값을 읽습니다. 실험의 SQLite `BEGIN IMMEDIATE`에서도 다른 연결의 일반 `SELECT`는 막히지 않고 옛 값 0을 읽었습니다. 그래서 같은 값을 고치는 모든 경로가 `FOR UPDATE`로 읽어야 lost update를 막을 수 있습니다.

**Q. 락을 오래 못 얻으면 어떻게 되나요?**  
InnoDB는 `innodb_lock_wait_timeout`(기본 50초)만큼 기다린 뒤 에러를 냅니다. 오래 기다리게 두기보다 `NOWAIT`로 바로 실패시키거나, 큐처럼 쓰는 테이블이라면 `SKIP LOCKED`로 잠긴 행을 건너뛰는 방법도 있습니다. 여러 행을 서로 다른 순서로 잠그면 데드락이 생길 수 있어서, 잠그는 순서를 정해 두는 것도 함께 봐야 합니다.

**Q. 재고 차감처럼 숫자를 빼는 경우에도 락이 필요한가요?**  
`UPDATE items SET qty = qty - 1 WHERE id = ? AND qty > 0`처럼 DB가 한 문장 안에서 검사와 계산을 하면, 따로 읽고 쓰는 구간이 없어서 락을 명시하지 않아도 됩니다. 읽은 값으로 복잡한 계산을 해야 할 때 비관적 락이나 낙관적 락을 씁니다(원자적 UPDATE 측정은 다음 질문에서 다룰 예정).

## 흔한 오답

- "비관적 락이 더 안전하고 빠릅니다" → 이 실험에서 유실은 두 방법 모두 0건이었고, 전체 시간은 같았습니다(2,767.2ms vs 2,771.7ms). 비관적 락이 나은 건 재시도와 긴 꼬리 지연이 없다는 점입니다.
- "비관적 락은 테이블을 잠급니다" → InnoDB의 `FOR UPDATE`는 테이블 전체가 아니라 검색하면서 지나간 인덱스 레코드를 잠급니다(기본 격리 수준에서는 그 앞의 간격까지). 다만 인덱스를 못 타면 훑은 행을 모두 잠그게 되고, 실험에서 락을 넓게 잡자 흩어진 요청도 2.49배 느려졌습니다.

## 참고

- [MySQL 8.4: Locking Reads (FOR UPDATE, NOWAIT, SKIP LOCKED)](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)
- [MySQL 8.4: Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html)
- [MySQL 8.4: innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.4/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)
- [Spring Data JPA: Locking](https://docs.spring.io/spring-data/jpa/reference/jpa/locking.html)
- [SQLite: BEGIN TRANSACTION (DEFERRED, IMMEDIATE, EXCLUSIVE)](https://www.sqlite.org/lang_transaction.html)
