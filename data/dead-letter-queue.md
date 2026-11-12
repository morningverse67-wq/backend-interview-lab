# DP-08. 처리할 수 없는 메시지는 어떻게 다루나요?

## 30초 답변

몇 번을 다시 해도 처리되지 않는 poison 메시지는 시도 횟수를 정해 두고, 넘으면 dead letter queue(DLQ)로 옮기겠습니다. 직접 재 보니 메시지 1만 건 중 poison이 129건일 때, 제자리에서 계속 재시도하면 첫 poison에서 소비가 멈춰 정상 메시지가 38건만 처리됐습니다. 최대 3번 시도 뒤 DLQ로 보내자 정상 메시지 9,871건이 모두 처리됐고 DLQ에는 poison 129건만 남았습니다.

## 조금 더 깊게

```python
if ok:
    ack(msg)
elif msg.tries >= 3:
    dlq.send(msg)   # 원인을 볼 수 있게 따로 보관
    ack(msg)        # 원래 큐에서는 빼기
else:
    retry_later(msg)
```

- 일시 오류는 다시 하면 성공하므로 재시도 횟수를 어느 정도 허용해야 DLQ가 깨끗합니다.
- DLQ에는 원인을 볼 수 있게 원본과 오류 내용을 같이 남기고, 알림을 걸어 둡니다.
- 원인을 고친 뒤 DLQ의 메시지를 원래 큐로 다시 넣는 절차도 필요합니다.

## 직접 측정한 근거

| 정책 | 쓴 시도 | 정상 처리 | 남은 큐 | DLQ | DLQ 속 정상 메시지 |
|---|---:|---:|---:|---:|---:|
| 제자리 무한 재시도 | 12,000 | 38 | 9,962 | 0 | 0 |
| 맨 뒤로 보내며 무한 재시도 | 12,000 | 9,871 | 129 | 0 | 0 |
| 재시도 없이 바로 DLQ | 10,000 | 9,357 | 0 | 643 | 514 |
| 최대 3번 시도 뒤 DLQ | 10,772 | 9,871 | 0 | 129 | 0 |

- 재현 코드: [backend-lab/033-dead-letter-queue](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/033-dead-letter-queue)

<sub>Python 3.13.12, macOS arm64. 시드 고정이라 매번 같은 수. 오류 비율과 시도 한도는 실험용 값.</sub>

## 꼬리 질문

**Q. 재시도 없이 바로 DLQ로 보내면 안 되나요?**  
큐는 막히지 않지만 일시 오류 514건까지 DLQ로 가서 정상 메시지가 섞입니다. 몇 번은 다시 해 보는 편이 DLQ를 깨끗하게 유지합니다.

**Q. 맨 뒤로 보내는 것만으로 충분하지 않나요?**  
정상 메시지는 모두 처리되지만, poison이 한도 없이 돌며 시도 횟수를 계속 가져갑니다(이 실험에서 1,615번). 횟수 제한과 DLQ가 있으면 그 낭비가 387번에서 끝납니다.

## 흔한 오답

- "실패하면 성공할 때까지 재시도합니다" → 항상 오류인 메시지 하나가 소비 전체를 멈췄습니다(정상 38건).
- "DLQ에 보내면 해결입니다" → DLQ를 열어 원인을 보고 다시 넣는 절차까지 있어야 합니다.

## 참고

- [AWS SQS: Dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Enterprise Integration Patterns: Dead Letter Channel](https://www.enterpriseintegrationpatterns.com/patterns/messaging/DeadLetterChannel.html)
