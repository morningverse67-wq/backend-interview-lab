# DP-06. DB 저장과 메시지 발행은 어떻게 같이 처리하나요?

## 30초 답변

DB와 메시지 브로커는 한 트랜잭션으로 묶이지 않아서, 저장과 발행 사이에 장애가 나면 한쪽만 반영됩니다. 그래서 이벤트를 같은 DB의 outbox 테이블에 주문과 같은 트랜잭션으로 저장하고, 릴레이가 그 테이블을 읽어 발행하게 하겠습니다. 직접 재 보니 주문 1만 건에서 단계 사이에 2% 확률로 죽게 했을 때, 커밋 후 발행은 이벤트 197건을 잃었고 발행 후 커밋은 없는 주문 이벤트 317건을 내보냈습니다. outbox는 둘 다 0이었지만 216건이 두 번 나가서, 소비자에서 이벤트 id로 중복을 거르는 처리를 같이 두겠습니다.

## 조금 더 깊게

```python
with db:  # 한 트랜잭션
    db.execute("INSERT INTO orders VALUES(?)", (order_id,))
    db.execute("INSERT INTO outbox(id, order_id) VALUES(?, ?)", (event_id, order_id))

# 릴레이
for event_id, order_id in db.execute("SELECT id, order_id FROM outbox WHERE sent=0"):
    broker.publish(event_id, order_id)   # 여기서 죽으면 다음에 다시 보냄
    db.execute("UPDATE outbox SET sent=1 WHERE id=?", (event_id,))
```

- 주문과 outbox 행은 같이 커밋되거나 같이 빠지므로, 저장된 주문의 이벤트는 언젠가 반드시 나갑니다.
- 발행 후 표시 전에 죽으면 다시 보내므로 전달은 최소 한 번(at-least-once)입니다. 정확히 한 번 처리는 소비자 멱등성으로 맞춥니다.
- 릴레이는 outbox를 주기적으로 읽는 방식(polling)이나 DB 변경 로그를 읽는 방식(CDC)으로 만듭니다.

## 직접 측정한 근거

| 방식 | 저장된 주문 | 발행된 이벤트 | 유실 | 유령 | 중복 |
|---|---:|---:|---:|---:|---:|
| 커밋 후 발행 | 9,889 | 9,692 | 197 | 0 | 0 |
| 발행 후 커밋 | 9,683 | 10,000 | 0 | 317 | 0 |
| outbox + 릴레이 | 9,905 | 10,121 | 0 | 0 | 216 |

- 소비자가 이벤트 id로 거르자 10,121번 받은 이벤트 중 9,905번만 처리해 저장된 주문 수와 같아졌습니다.
- 재현 코드: [backend-lab/031-transactional-outbox](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/031-transactional-outbox)

<sub>Python 3.13.12, macOS arm64. 시드 고정이라 매번 같은 수. 장애 확률(커밋 실패 1%, 단계 사이 죽기 2%)은 실험용 값.</sub>

## 꼬리 질문

**Q. 커밋 후 발행하고, 실패하면 재시도하면 되지 않나요?**  
발행 호출이 실패로 돌아오면 재시도할 수 있지만, 커밋 직후 프로세스가 죽으면 재시도할 주체가 없습니다. 실험의 유실 197건이 그 경우입니다. 보낼 일을 DB에 남겨 두어야 재시작 후에도 이어서 보낼 수 있습니다.

**Q. outbox면 중복은 왜 생기나요?**  
브로커에 보낸 것과 `sent=1` 표시도 서로 다른 시스템이라, 그 사이에 죽으면 보냈다는 기록이 남지 않습니다. 실험에서 216건이 다시 나갔고, 소비자 쪽 이벤트 id 기록으로 걸렀습니다.

## 흔한 오답

- "발행을 먼저 하면 유실이 없으니 안전합니다" → 유실은 0이지만 DB에 없는 주문 이벤트가 317건 나갔습니다.
- "outbox를 쓰면 정확히 한 번 전달됩니다" → 전달은 최소 한 번이고, 실험에서 216건이 중복됐습니다.

## 참고

- [microservices.io: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)
- [Debezium: Outbox Event Router](https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html)
- [Python 문서: sqlite3](https://docs.python.org/3/library/sqlite3.html)
