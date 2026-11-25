# DP-10. 삭제한 데이터를 남겨 두면 유니크 제약은 어떻게 거나요?

## 30초 답변

살아 있는 행에만 거는 부분 유니크 인덱스(`WHERE deleted_at IS NULL`)를 쓰겠습니다. 직접 재 보니 10만 행 중 9만 행이 soft delete된 테이블에서, 전체 `UNIQUE(email)`은 탈퇴한 이메일 1,000개의 재가입을 모두 오류로 막았고, 부분 유니크 인덱스는 재가입 오류 0이면서 활동 중인 이메일 중복 1,000건은 똑같이 다 막았습니다. 인덱스 크기도 757페이지에서 74페이지로 줄었습니다.

## 조금 더 깊게

```sql
CREATE UNIQUE INDEX ux_users_email ON users(email) WHERE deleted_at IS NULL;
SELECT id FROM users WHERE email = ? AND deleted_at IS NULL;
```

- 전체 유니크는 지운 행도 "사용 중"으로 봐서 재가입을 막습니다.
- 부분 인덱스는 지운 행을 담지 않아 삭제 비율이 높을수록 작아집니다.
- 쿼리에 인덱스와 같은 조건(`deleted_at IS NULL`)이 있어야 그 인덱스를 씁니다.

## 직접 측정한 근거

| 방식 | 인덱스 페이지 | 재가입 오류 | 활동 중 중복 막음 | 조회 1만 건 ms |
|---|---:|---:|---:|---:|
| 전체 UNIQUE(email) | 757 | 1,000 | 1,000 | 8.1 |
| 부분 UNIQUE(email) WHERE deleted_at IS NULL | 74 | 0 | 1,000 | 6.0 |

- 재현 코드: [backend-proof/035-soft-delete-partial-index](https://github.com/morningverse67-wq/backend-proof/tree/main/labs/035-soft-delete-partial-index)

<sub>Python 3.13.12, SQLite 3.50.4, macOS arm64. 페이지 수와 오류 건수는 매번 같고 ms는 실행마다 조금씩 달라집니다(7.9~8.2 대 5.9~6.4).</sub>

## 꼬리 질문

**Q. MySQL에서도 같은 방법을 쓸 수 있나요?**  
MySQL은 부분 인덱스가 없어서, 삭제되면 NULL이 되는 생성 컬럼에 유니크를 거는 식으로 비슷하게 만듭니다.

**Q. 이 수치를 실제 서비스에 그대로 쓸 수 있나요?**  
아닙니다. SQLite 인메모리 실험이라 실제 DB 서버의 크기와 속도는 다를 수 있습니다. 재가입·중복 결과는 제약의 동작이라 같습니다.

## 흔한 오답

- "이메일에 UNIQUE만 걸면 됩니다" → soft delete와 함께 쓰면 탈퇴한 사람이 같은 이메일로 다시 가입할 수 없습니다.

## 참고

- [PostgreSQL: Partial Indexes](https://www.postgresql.org/docs/current/indexes-partial.html)
- [SQLite: Partial Indexes](https://www.sqlite.org/partialindex.html)
