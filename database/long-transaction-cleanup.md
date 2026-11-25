# DP-12. 오래 열린 트랜잭션이 왜 문제인가요?

## 30초 답변

오래 열린 트랜잭션은 DB가 옛 버전 행을 정리하지 못하게 막습니다. 직접 재 보니 1만 행에 1,000건씩 20번 UPDATE할 때, 긴 읽기가 없으면 정리가 20번 모두 돼 WAL이 0 MB로 돌아갔고, 읽기 트랜잭션 하나를 열어 두면 20번 모두 정리가 막혀 17.65 MB까지 쌓였습니다. 그 트랜잭션을 닫자 다음 정리에서 바로 0 MB가 됐습니다.

## 조금 더 깊게

- MVCC에서는 아직 옛 버전을 볼 수 있는 트랜잭션이 있으면 그 버전을 지울 수 없습니다. PostgreSQL은 VACUUM, MySQL InnoDB는 purge, SQLite는 WAL checkpoint가 그 앞에서 멈춥니다.
- 오래 열린 읽기는 계속 시작 시점 스냅샷을 봅니다. 실험에서도 끝까지 잔액 합계 10,000,000을 봤습니다(실제는 10,020,000).
- 찾기: PostgreSQL `pg_stat_activity`에서 `xact_start`가 오래된 세션, 특히 `idle in transaction`.
- 막기: 배치·리포트는 트랜잭션을 짧게 끊어 읽고, `idle_in_transaction_session_timeout`·`statement_timeout`을 겁니다.

## 직접 측정한 근거

| 상황 | 정리 실패(busy) | WAL 최대 MB | 마지막 WAL MB |
|---|---:|---:|---:|
| 긴 읽기 없음 | 0/20 | 0.00 | 0.00 |
| 긴 읽기를 10번째 묶음 뒤에 닫음 | 9/20 | 7.94 | 0.00 |
| 긴 읽기가 끝까지 열려 있음 | 20/20 | 17.65 | 17.65 |

- 재현 코드: [backend-proof/037-long-transaction-wal-growth](https://github.com/morningverse67-wq/backend-proof/tree/main/labs/037-long-transaction-wal-growth)

<sub>Python 3.13.12, SQLite 3.50.4, macOS arm64. 같은 시드라 여러 번 돌려도 숫자가 같았습니다.</sub>

## 꼬리 질문

**Q. 읽기만 하는 트랜잭션인데도 문제가 되나요?**  
네. 쓰기를 하지 않아도 스냅샷을 잡고 있으면 그 시점 이후의 옛 버전이 남습니다. 실험의 긴 트랜잭션도 SELECT 한 번만 했습니다.

**Q. 이 수치를 PostgreSQL에 그대로 쓸 수 있나요?**  
SQLite WAL 파일 크기로 잰 결과라 수치는 다릅니다. PostgreSQL에서는 dead tuple 수와 테이블 크기(bloat)로 나타나고, 막히는 원리는 같습니다.

## 흔한 오답

- "VACUUM을 더 자주 돌리면 됩니다" → 오래 열린 트랜잭션이 남아 있으면 VACUUM을 돌려도 그 뒤의 옛 버전은 지우지 못합니다. 먼저 그 트랜잭션을 찾아 닫습니다.

## 참고

- [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html#avoiding_excessively_large_wal_files)
- [PostgreSQL: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
