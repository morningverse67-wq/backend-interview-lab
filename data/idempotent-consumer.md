# DP-07. 같은 메시지가 두 번 오면 소비자는 어떻게 처리하나요?

## 30초 답변

대부분의 브로커는 최소 한 번 전달이라 같은 메시지가 다시 올 수 있습니다. 그래서 소비자에 처리한 이벤트 id를 기록하는 inbox 테이블을 두고, id 기록과 실제 반영을 한 트랜잭션에서 하겠습니다. 직접 재 보니 이벤트 1만 건에 재전송 3%, 반영 직후 죽기 2%를 넣었을 때 그냥 반영은 548건, 반영 뒤 id를 따로 기록한 방식도 223건을 더 적립했습니다. 한 트랜잭션으로 묶자 초과 반영이 0이었습니다.

## 조금 더 깊게

```python
try:
    with db:  # 한 트랜잭션
        db.execute("INSERT INTO inbox(event_id) VALUES(?)", (event_id,))  # 기본키
        db.execute("UPDATE account SET point = point + 100 WHERE id=?", (user_id,))
except sqlite3.IntegrityError:
    pass  # 이미 처리한 이벤트: ack만 보냄
ack(event_id)
```

- 기본키 충돌이 "처리함"을 알려 주므로, 조회 후 처리처럼 두 단계로 나눌 필요가 없습니다.
- 반영 대상이 다른 시스템(결제 API 등)이면 같은 트랜잭션에 못 넣으니, 그쪽에 멱등성 키를 넘깁니다.
- inbox는 브로커가 다시 보낼 수 있는 기간만큼 보관하고 정리합니다.

## 직접 측정한 근거

| 방식 | 받은 횟수 | 반영 횟수 | 초과 반영 |
|---|---:|---:|---:|
| 그냥 반영 | 10,548 | 10,548 | 548 |
| 반영 후 id 기록(따로 커밋) | 10,533 | 10,223 | 223 |
| inbox + 반영 한 트랜잭션 | 10,533 | 10,000 | 0 |

- 재현 코드: [backend-lab/032-idempotent-consumer](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/032-idempotent-consumer)

<sub>Python 3.13.12, macOS arm64. 시드 고정이라 매번 같은 수. 재전송 3%, 반영 뒤 죽기 2%는 실험용 값.</sub>

## 꼬리 질문

**Q. 처리한 id를 기록하고 있는데도 왜 223건이 중복됐나요?**  
반영 커밋과 id 기록 커밋 사이에 죽으면 반영은 남고 기록은 없습니다. ack도 안 갔으니 같은 이벤트가 다시 오고, 기록이 없어서 또 반영됩니다. 둘을 한 트랜잭션에 넣어야 이 틈이 사라집니다.

**Q. 브로커의 exactly-once 설정을 쓰면 되지 않나요?**  
브로커 안에서의 중복은 줄일 수 있지만, 소비자가 DB에 반영한 뒤 죽는 경우까지 브로커가 알 수는 없습니다. 결국 반영하는 쪽에서 멱등하게 만드는 것이 확실합니다.

## 흔한 오답

- "처리한 id만 기록하면 중복이 없습니다" → 기록을 따로 커밋하면 223건이 중복 반영됐습니다.
- "중복은 드무니 무시해도 됩니다" → 재전송 3%, 죽기 2%만으로 5.48%가 더 적립됐습니다.

## 참고

- [microservices.io: Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html)
- [Apache Kafka 문서: Message Delivery Semantics](https://kafka.apache.org/documentation/#semantics)
- [Python 문서: sqlite3](https://docs.python.org/3/library/sqlite3.html)
