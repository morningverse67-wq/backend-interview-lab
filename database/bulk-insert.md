# DB-07. 대량 INSERT를 빠르게 하는 방법은?

## 30초 답변

가장 먼저 볼 것은 COMMIT 횟수입니다. 한 건마다 커밋하면 커밋마다 로그를 디스크에 확실히 기록할 때까지 기다리기 때문에, 아래 실험에서는 같은 1만 건이 1,948.84ms에서 트랜잭션 하나로 묶자 8.08ms(241.2배)가 됐습니다.
그다음으로 `executemany`/JDBC batch나 여러 행을 한 문장에 넣는 `VALUES (...), (...)`로 문장 수와 네트워크 왕복을 줄입니다.
다만 전부 한 트랜잭션으로 묶으면 한 건만 실패해도 전부 롤백되고 락도 오래 잡으니, 보통 1,000건 정도의 단위로 나눠 커밋하고 실패하면 어디서부터 다시 넣을지 정해 둡니다.

## 조금 더 깊게

```sql
-- 느린 방식: 문장마다 자동 커밋
INSERT INTO orders (...) VALUES (?, ?, ?, ?);   -- COMMIT
INSERT INTO orders (...) VALUES (?, ?, ?, ?);   -- COMMIT ... x10,000

-- 묶는 방식: 트랜잭션 하나 + 여러 행 VALUES
BEGIN;
INSERT INTO orders (...) VALUES (?, ?, ?, ?), (?, ?, ?, ?), ...;  -- 500건씩
COMMIT;
```

- **COMMIT이 비싼 이유**: 커밋이 끝났다고 응답하려면 변경 내용이 디스크에 남았다고 보장해야 합니다. 그래서 커밋마다 로그(SQLite는 저널, MySQL InnoDB는 redo log)를 쓰고 fsync로 기다립니다. 커밋 횟수가 곧 디스크 동기화 횟수입니다.
- **문장을 묶는 이유**: MySQL처럼 서버와 통신하는 DB는 문장마다 네트워크 왕복이 생깁니다. JDBC `addBatch()`/`executeBatch()`에 MySQL Connector/J의 `rewriteBatchedStatements=true`를 켜면 여러 INSERT를 여러 행 `VALUES` 한 문장으로 바꿔 보냅니다.
- **더 큰 적재**: 수십만 건 이상이면 MySQL `LOAD DATA`, PostgreSQL `COPY`처럼 대량 적재 전용 명령이 있고, 적재 중에는 인덱스를 나중에 만드는 방법도 씁니다.
- **커밋 단위**: 크게 잡을수록 빠르지만 금방 평평해집니다. 실패 시 잃는 범위와 락을 잡는 시간을 같이 보고 정합니다.

## 직접 측정한 근거

![1만 건 INSERT가 한 건마다 COMMIT 하면 1,948.84ms, 트랜잭션 하나로 묶으면 8.08ms로 241.2배, executemany 6.52ms, 500건씩 VALUES 6.27ms. 1,000건마다 커밋하면 10.16ms. 7,501번째 행에서 실패하면 한 트랜잭션은 0건, 1,000건 단위는 7,000건이 남는 비교 그래프](../assets/db-07-bulk-insert.png)

주문 1만 건, `order_no` UNIQUE + `user_id` 인덱스 1개, SQLite 파일 DB(`journal_mode=delete`, `synchronous=FULL`). 메모리 DB는 커밋 때 디스크 동기화가 없어서 파일 DB로 쟀고, 방법마다 빈 DB에서 시작해 결과(건수, 합계, 최소/최대 주문번호)가 같은지 확인.

| 방법 (1만 건 전체) | COMMIT | 시간 (ms) | 매번 COMMIT 대비 |
|---|---:|---:|---:|
| 한 건씩 + 매번 COMMIT | 10,000 | 1,948.84 | 1.0x |
| 한 건씩, 트랜잭션 1개 | 1 | 8.08 | 241.2x |
| `executemany`, 트랜잭션 1개 | 1 | 6.52 | 299.1x |
| `VALUES` 500건씩, 트랜잭션 1개 | 1 | 6.27 | 310.8x |

| 몇 건마다 COMMIT (`executemany`) | COMMIT 횟수 | 시간 (ms) | 7,501번째 행에서 실패하면 남는 행 |
|---:|---:|---:|---:|
| 1 | 10,000 | 1,917.89 | - |
| 10 | 1,000 | 323.20 | - |
| 100 | 100 | 49.04 | 7,500 |
| 1,000 | 10 | 10.16 | 7,000 |
| 10,000 | 1 | 6.63 | 0 |

- 차이의 대부분은 COMMIT 횟수에서 나왔습니다. 트랜잭션을 묶은 뒤 `executemany`와 여러 행 `VALUES`의 추가 이득은 8.08ms → 6.27ms 정도로 작았는데, SQLite는 같은 프로세스 안의 DB라 네트워크 왕복이 없기 때문입니다. 서버형 DB에서의 왕복 비용은 이 실험에서 재지 않았습니다.
- 1,000건 단위 커밋(10.16ms)은 전부 하나로 묶은 것(6.63ms)과 차이가 작았고, 실패해도 앞의 7,000건은 남았습니다.
- 재현 코드: [backend-lab/008-bulk-insert](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/008-bulk-insert)

<sub>Python 3.13 / SQLite 3.50, macOS 로컬 SSD, 3회 중 최소값. COMMIT 비용은 디스크와 OS 설정에 따라 크게 달라지므로 수치는 환경에 따라 달라질 수 있음.</sub>

## 꼬리 질문

**Q. 그럼 항상 전부 한 트랜잭션으로 묶는 게 좋나요?**  
아닙니다. 위 실험에서 7,501번째 행 하나가 UNIQUE를 어기자 한 트랜잭션은 전부 롤백돼 0건이 남았습니다. 트랜잭션이 길면 그동안 락과 undo 로그도 오래 유지됩니다. 전부 성공하거나 전부 취소돼야 하는 데이터가 아니라면 1,000건 정도로 나누는 편이 속도 손해도 작습니다(10.16ms vs 6.63ms).

**Q. 나눠서 커밋하다 중간에 실패하면 어떻게 하나요?**  
이미 커밋된 부분은 남기 때문에(1,000건 단위에서 7,000건) 다시 돌릴 때 중복이 생기지 않게 해야 합니다. 마지막으로 커밋한 위치를 기록해 이어서 넣거나, 주문번호 같은 UNIQUE 키를 두고 `INSERT ... ON DUPLICATE KEY UPDATE`(MySQL), `ON CONFLICT DO NOTHING`(PostgreSQL)처럼 다시 넣어도 결과가 같게 만듭니다.

**Q. JPA로 `saveAll()`을 하면 배치 INSERT가 되나요?**  
설정 없이는 한 건씩 나갑니다. `hibernate.jdbc.batch_size`를 켜야 JDBC batch로 묶이고, 키 생성 전략이 `IDENTITY`면 INSERT마다 생성된 ID를 받아야 해서 Hibernate가 INSERT 배치를 하지 않습니다. 이 부분은 이 실험에서 재지 않았습니다.

## 흔한 오답

- "`executemany`(batch)만 쓰면 빨라집니다" → 이 실험에서 이득의 대부분은 COMMIT을 줄인 데서 나왔습니다. 문장을 묶어도 autocommit으로 매번 커밋하면 커밋 비용은 그대로입니다.
- "트랜잭션은 크면 클수록 좋습니다" → 1,000건 단위와 1만 건 한 번의 차이는 3.53ms였지만, 실패 시 남는 행은 7,000건과 0건으로 갈렸습니다. 속도만 보고 단위를 정하면 안 됩니다.

## 참고

- [MySQL 8.0: Optimizing INSERT Statements](https://dev.mysql.com/doc/refman/8.0/en/insert-optimization.html)
- [MySQL Connector/J: Performance Extensions (rewriteBatchedStatements)](https://dev.mysql.com/doc/connector-j/en/connector-j-connp-props-performance-extensions.html)
- [PostgreSQL: Populating a Database](https://www.postgresql.org/docs/current/populate.html)
- [SQLite FAQ: INSERT is really slow](https://www.sqlite.org/faq.html#q19)
