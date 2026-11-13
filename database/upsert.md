# DP-09. 없으면 넣고 있으면 고치는 작업은 어떻게 처리하나요?

## 30초 답변

`INSERT ... ON CONFLICT DO UPDATE`처럼 판단과 쓰기를 한 문장으로 처리하겠습니다. 직접 재 보니 같은 새 키에 요청 2개가 동시에 도착하는 1,000쌍에서, "조회 후 INSERT/UPDATE"는 중복 키 오류 1,000번과 사라진 증가분 1,000을 냈고 ON CONFLICT는 둘 다 0이었습니다. 같은 요청 10만 건은 98ms 대 45ms였습니다.

## 조금 더 깊게

```sql
INSERT INTO counter (k, n) VALUES (?, 1)
ON CONFLICT (k) DO UPDATE SET n = n + 1;
```

- 조회와 쓰기 사이에 다른 요청이 끼어들 수 있어서, 조회 후 쓰기는 둘 다 "없음"을 보는 순간이 생깁니다.
- 충돌을 판단할 기준이 되는 유니크 키나 기본 키가 있어야 합니다.
- 문법은 DB마다 다릅니다(PostgreSQL·SQLite는 ON CONFLICT, MySQL은 ON DUPLICATE KEY UPDATE).

## 직접 측정한 근거

| 방식 | 중복 키 오류 | 사라진 증가분 | 10만 건 ms |
|---|---:|---:|---:|
| 조회 후 INSERT/UPDATE | 1,000 | 1,000 | 98 |
| INSERT ... ON CONFLICT | 0 | 0 | 45 |

- 재현 코드: [backend-lab/034-upsert-on-conflict](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/034-upsert-on-conflict)

<sub>Python 3.13.12, SQLite 3.50.4, macOS arm64. 경합은 순서를 고정해 재현했고 ms는 실행마다 달라집니다(같은 코드 여러 번: 98~148 대 45~62).</sub>

## 꼬리 질문

**Q. 오류를 잡아서 UPDATE로 다시 하면 되지 않나요?**  
정확성은 맞출 수 있지만 오류 처리와 재시도 코드가 늘어납니다. 한 문장으로 끝나는 쪽이 짧고 왕복도 적습니다.

**Q. 이 실험 결과를 실제 서비스에 그대로 쓸 수 있나요?**  
아닙니다. 한 프로세스에서 순서를 고정해 재현한 것이라, 실제 DB 서버와 다중 연결에서는 수치가 다를 수 있습니다.

## 흔한 오답

- "먼저 조회해서 없을 때만 넣으면 안전합니다" → 조회와 쓰기 사이에 다른 요청이 끼어들 수 있습니다.

## 참고

- [PostgreSQL: INSERT ... ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html)
- [SQLite: UPSERT](https://www.sqlite.org/lang_upsert.html)
