# DP-13. 대량으로 DELETE했는데 디스크가 그대로예요. 왜 그런가요?

## 30초 답변

DELETE는 행을 지우고 빈 페이지를 DB 안의 재사용 목록(freelist)에 올려 둘 뿐, 파일을 OS에 돌려주지 않습니다. 직접 재 보니 로그 20만 행 중 18만 행을 지워도 파일은 27.70 MB 그대로였고 빈 페이지가 6,373개 생겼습니다. 그 자리는 다음 INSERT가 먼저 써서, 18만 행을 다시 넣어도 27.91 MB였습니다. 파일을 줄이려면 `VACUUM`을 돌려야 했고, 그러면 2.77 MB가 됐습니다.

## 조금 더 깊게

- 지우고 다시 쌓이는 테이블이면 빈 공간이 재사용되므로 파일을 줄일 필요가 크지 않습니다.
- 정말 공간을 돌려받아야 하면 재작성이 필요합니다. SQLite는 `VACUUM` 또는 `auto_vacuum=INCREMENTAL` + `incremental_vacuum`(실험에서 2.81 MB), PostgreSQL은 `VACUUM FULL`이나 pg_repack입니다.
- PostgreSQL 일반 `VACUUM`은 공간을 재사용 가능하게 표시하는 일이라 파일 크기는 거의 줄지 않습니다. `VACUUM FULL`은 테이블을 잠그므로 운영 중에는 조심합니다.
- 오래된 로그를 주기적으로 지운다면 날짜별 파티션을 두고 파티션째 떼어 내는 편이 DELETE보다 가볍습니다.

## 직접 측정한 근거

| 방법 | DELETE 전 MB | DELETE 뒤 MB (빈 페이지) | 정리 뒤 MB (빈 페이지) |
|---|---:|---:|---:|
| DELETE만 | 27.70 | 27.70 (6,373) | 27.70 (6,373) |
| DELETE 뒤 VACUUM | 27.70 | 27.70 (6,373) | 2.77 (0) |
| auto_vacuum=INCREMENTAL + incremental_vacuum | 27.74 | 27.74 (6,373) | 2.81 (0) |

- 재현 코드: [backend-lab/038-bulk-delete-freelist-vacuum](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/038-bulk-delete-freelist-vacuum)

<sub>Python 3.13.12, SQLite 3.50.4, macOS arm64. 데이터가 고정이라 여러 번 돌려도 숫자가 같았습니다.</sub>

## 꼬리 질문

**Q. 그럼 대량 삭제 뒤에는 항상 VACUUM을 돌려야 하나요?**  
아니요, 다시 데이터가 쌓이는 테이블이면 빈 페이지가 재사용됩니다. 실험에서도 다시 넣었을 때 페이지는 7,092개에서 7,145개로 53개만 늘었습니다. 디스크를 실제로 돌려받아야 할 때만 재작성합니다.

**Q. auto_vacuum을 켜 두면 VACUUM이 필요 없나요?**  
SQLite `auto_vacuum`은 테이블을 만들기 전에 정해야 하고(나중에 바꾸면 `VACUUM` 한 번 필요), INCREMENTAL 모드는 `incremental_vacuum`을 불러야 줄어듭니다. 페이지를 옮기기만 해서 조각화는 남을 수 있습니다.

## 흔한 오답

- "DELETE가 안 된 것 같습니다" → 행은 지워졌습니다(실험에서 남은 행 2만 개, `integrity_check` ok). 파일 크기만 그대로인 것입니다.

## 참고

- [SQLite: VACUUM](https://www.sqlite.org/lang_vacuum.html)
- [SQLite: PRAGMA auto_vacuum](https://www.sqlite.org/pragma.html#pragma_auto_vacuum)
- [PostgreSQL: Routine Vacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html)
