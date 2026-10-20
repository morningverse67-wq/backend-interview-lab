# TX-01. Lost update는 언제 생기고 낙관적 락으로 어떻게 막나요?

## 30초 답변

Lost update는 두 요청이 같은 값을 읽고, 각자 애플리케이션에서 계산한 값으로 덮어쓸 때 생깁니다. 먼저 쓴 변경이 에러 없이 사라집니다. 아래 실험에서 한 행에 `+1` 요청을 1,600번 동시에 보냈더니 최종 값이 200이었습니다.
낙관적 락은 행에 `version` 컬럼을 두고 `UPDATE ... WHERE id = ? AND version = ?`처럼 읽을 때 본 버전일 때만 쓰게 합니다. 바뀐 행이 0이면 누가 먼저 고친 것이니 다시 읽고 재시도하거나 에러를 돌려줍니다. 같은 실험에서 유실은 0건이 됐지만, 한 행에 요청이 몰리자 재시도가 5,995번 생기고 전체 시간이 2.09배가 됐습니다. 그래서 충돌이 드문 곳에 씁니다.

## 조금 더 깊게

```sql
SELECT qty, version FROM items WHERE id = 1;           -- qty 10, version 3

UPDATE items
   SET qty = 11, version = version + 1
 WHERE id = 1 AND version = 3;                          -- 영향받은 행 1: 성공 / 0: 충돌
```

- **왜 생기나**: `SELECT`와 `UPDATE`가 따로 실행되고, 그 사이에 다른 요청이 같은 행을 고칠 수 있습니다. `UPDATE`가 계산된 값(`qty = 11`)으로 덮어쓰면 그사이의 변경은 흔적 없이 사라집니다.
- **낙관적 락이 막는 방법**: 검사(`version = 3`)와 쓰기가 한 `UPDATE` 문 안에서 일어나므로, DB가 그 행에 쓰는 순간 버전이 같은지 봅니다. 락을 미리 잡지 않고, 충돌은 나중에 "바뀐 행 0개"로 알게 됩니다.
- **JPA에서는**: 엔티티에 `@Version` 필드를 두면 Hibernate가 같은 `WHERE version = ?`를 만들어 주고, 바뀐 행이 0이면 예외(`OptimisticLockException` 계열)를 던집니다. 재시도는 자동이 아니라서 직접 정해야 합니다.
- **언제 안 맞나**: 같은 행에 요청이 몰리면 대부분이 충돌하고 재시도합니다. 이런 곳은 비관적 락(`SELECT ... FOR UPDATE`)으로 줄을 세우거나, `SET qty = qty + 1`처럼 DB가 한 문장 안에서 계산하게 합니다.

## 직접 측정한 근거

![한 행에 +1 요청 1,600번을 보내면 락 없이는 최종 값 200(1,400건 유실), 낙관적 락은 1,600(유실 0건)이지만 재시도 5,995번에 전체 시간 1,319.4ms에서 2,754.8ms(2.09배). 1,000행에 흩어지면 락 없이도 10건 유실, 낙관적 락은 재시도 10번에 1.03배인 비교 그래프](../assets/tx-01-optimistic-lock.png)

SQLite 파일 DB(WAL), 8스레드 x 200번 = `+1` 요청 1,600번. 요청마다 `SELECT` → 1ms 대기(애플리케이션 처리 시간 흉내) → `UPDATE`. 낙관적 락은 바뀐 행이 0이면 다시 읽고 재시도(횟수 제한 없음). 3회 실행의 지표별 중앙값이고, 낙관적 락은 매번 최종 값 1,600과 `version` 합계 1,600을 확인.

| 경우 | 방법 | 최종 값 (기대 1,600) | 사라진 요청 | 충돌(재시도) | 한 요청 최대 재시도 | 전체 시간 (ms) |
|---|---|---:|---:|---:|---:|---:|
| 한 행에 몰림 | 락 없음 | 200 | 1,400 | - | - | 1,319.4 |
| 한 행에 몰림 | 낙관적 락 | 1,600 | 0 | 5,995 | 124 | 2,754.8 |
| 1,000행에 흩어짐 | 락 없음 | 1,590 | 10 | - | - | 1,170.5 |
| 1,000행에 흩어짐 | 낙관적 락 | 1,600 | 0 | 10 | 1 | 1,200.6 |

- 충돌이 드물면(1,000행에 흩어짐) 낙관적 락은 재시도 10번, 시간 1.03배로 거의 비용이 없었습니다. 그래도 락이 없을 때는 10건이 사라졌습니다.
- 한 행에 몰리면 요청 1건당 평균 3.7번 재시도했고, 재시도를 3번으로 제한했다면 270건(16.9%)이 실패로 끝났을 것입니다.
- 재현 코드: [backend-lab/010-lost-update-optimistic-lock](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/010-lost-update-optimistic-lock)

<sub>Python 3.13 / SQLite 3.50, 파일 DB(WAL, synchronous=NORMAL), 3회 실행의 지표별 중앙값. 스레드 실행 순서에 따라 매번 조금씩 달라지고, 환경에 따라 수치는 달라질 수 있음.</sub>

## 꼬리 질문

**Q. 트랜잭션으로 감싸면 lost update가 안 생기지 않나요?**  
읽기와 쓰기를 따로 보내는 구조라면 트랜잭션만으로는 부족할 수 있습니다. 예를 들어 MySQL InnoDB의 기본 격리 수준에서 일반 `SELECT`는 잠금 없이 스냅샷을 읽으므로, 두 트랜잭션이 같은 값을 읽고 각자 덮어쓸 수 있습니다. 읽을 때 `SELECT ... FOR UPDATE`로 잠그거나, 버전을 검사하거나, 한 `UPDATE` 문 안에서 계산해야 합니다(이 실험에서 MySQL은 재지 않음).

**Q. 충돌이 나면 무조건 재시도하면 되나요?**  
아닙니다. 이 실험에서 한 행에 몰린 경우 가장 운 나쁜 요청은 124번 재시도했습니다. 보통 재시도 횟수와 대기 시간을 제한하고, 넘으면 "다른 사용자가 먼저 수정했습니다" 같은 에러로 돌려줍니다. 사용자가 직접 입력한 내용을 덮어쓰는 화면이라면 자동 재시도보다 다시 확인하게 하는 편이 맞습니다.

**Q. 낙관적 락과 비관적 락은 어떻게 고르나요?**  
충돌 빈도로 고릅니다. 충돌이 드물면 낙관적 락이 락을 기다리는 시간 없이 거의 그대로의 속도를 냅니다(이 실험 1.03배). 같은 행에 요청이 몰리면 재시도 비용이 커지므로 비관적 락이나 원자적 `UPDATE`를 검토합니다.

## 흔한 오답

- "낙관적 락은 락이 없으니 항상 더 빠릅니다" → 충돌이 많으면 다시 읽고 다시 쓰는 비용이 쌓입니다. 이 실험에서 한 행에 몰리자 2.09배 느려졌습니다.
- "버전을 읽어서 애플리케이션에서 비교한 뒤 UPDATE하면 됩니다" → 비교와 쓰기 사이에 또 다른 요청이 끼어들 수 있습니다. 버전 검사는 `UPDATE ... WHERE version = ?`처럼 쓰기 문장 안에 있어야 합니다.

## 참고

- [Hibernate ORM 6.6 User Guide: Optimistic locking](https://docs.jboss.org/hibernate/orm/6.6/userguide/html_single/Hibernate_User_Guide.html#locking-optimistic)
- [Spring Data JPA: Locking](https://docs.spring.io/spring-data/jpa/reference/jpa/locking.html)
- [MySQL 8.4: Consistent Nonlocking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-consistent-read.html)
- [MySQL 8.4: Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)
- [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html)
