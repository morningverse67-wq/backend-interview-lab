# API-01. Rate limiter 알고리즘별 차이는?

## 30초 답변

Fixed window는 고정 구간마다 카운트를 초기화해서 단순하지만, 창 경계에 요청이 몰릴 수 있습니다. Exact sliding window log는 직전 구간에 허용한 요청 시각을 보관해 어느 시점에서든 한도를 지키는 대신 로그 공간이 필요합니다. Token bucket은 일정한 속도로 토큰을 충전하고 용량만큼 burst를 허용합니다. 그래서 초당 충전율이 같아도 strict rolling-window 제한과 같지 않습니다. 먼저 구간별 쿼터인지, 직전 구간의 엄격한 상한인지, burst 허용인지 요구사항부터 정하겠습니다.

## 조금 더 깊게

- **Fixed window**: 실험은 `[k×1000, (k+1)×1000)`마다 허용 요청을 5개까지 셉니다. 경계 전후에 각 창의 몫을 쓸 수 있습니다. 상태는 키당 O(1)입니다.
- **Exact sliding window log**: `(t-1000, t]`의 허용 시각을 deque에 보관합니다. 정확히 1000ms 전의 시각은 제거하고 거절은 기록하지 않습니다. 제한 L에 대해 키당 O(L) 공간이며, 제거 작업은 최악 O(L), amortized O(1)입니다. 앞뒤 두 카운터에 가중치를 주는 sliding counter 근사와 구별해야 합니다.
- **Token bucket**: 용량 5, 충전율 5/s, 초기 가득 찬 상태에서 시작합니다. 요청당 1토큰을 차감하고 부족하면 즉시 거절합니다. 정수 credit으로 부분 토큰을 보존했습니다. 상태는 키당 O(1)입니다. 용량 B와 충전율 r은 길이 Δ인 구간에서 B+rΔ 예산을 뜻하지, 매 rolling 구간에서 rΔ개만 허용한다는 뜻이 아닙니다. 큐에 넣어 일정 간격으로 내보내는 구현도 아닙니다.

## 직접 측정한 근거

![정책별 요청 수, 허용 수, 거절 수와 rolling 1초 최대 허용 수. 경계 burst에서는 fixed window가 10개, 재충전 burst에서는 token bucket이 9개를 허용했습니다.](../assets/api-01-rate-limiter.svg)

2026-10-31, Python 3.13.12 / macOS(Darwin 25.6.0) / arm64에서 실행했습니다. 단일 키의 요청 시각을 정수 ms fake clock에 넣었습니다. 같은 시나리오는 세 정책에 동일하게 재생하며 매번 새 상태로 시작합니다. 공통 제한은 1000ms당 5개, token bucket 용량은 5, 충전율은 5/s입니다.

아래 셀은 **허용 수 / 직전 1초 최대 허용 수**입니다. 최대치는 모든 `(t-1000, t]`의 허용 요청을 센 값입니다.

| 동일 요청 목록 | 요청 수 | Fixed window | Exact sliding log | Token bucket |
|---|---:|---:|---:|---:|
| 0~2900ms, 100ms 간격 | 30 | 15 / 5 | 15 / 5 | 19 / 9 |
| 999ms에 5개 + 1000ms에 5개 | 10 | 10 / 10 | 5 / 5 | 5 / 5 |
| 0ms에 5개 + 800ms에 5개 | 10 | 5 / 5 | 5 / 5 | 9 / 9 |
| 0ms에 10개 + 10000ms에 10개 | 20 | 10 / 5 | 10 / 5 | 10 / 5 |

Token bucket은 800ms 동안 4토큰이 생겨 0ms의 5개에 더해 4개를 허용했습니다. Exact log는 0ms의 기록이 아직 남아 두 번째 묶음을 전부 거절했습니다. 유휴 후에도 token bucket에는 용량 5개만 쌓였습니다.

- 재현 코드와 전체 결과: [backend-lab/020-rate-limiter](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/020-rate-limiter)
- 실행: `python3 main.py` (표준 라이브러리만 사용)
- 검증: `regression_checks=26 PASS`, `workload_expectations=12 PASS`. 창 경계, 정확한 로그 만료, 거절 뒤 만료, 부분 토큰 누적, 유휴 용량 상한 등을 assert로 검사했습니다.
- 이것은 결정적 순차 시뮬레이션입니다. 동시성·분산 처리·네트워크·실제 latency·처리량은 측정하지 않았습니다. 알고리즘 공간 복잡도도 실측 메모리 값이 아닙니다.

## 꼬리 질문

**Q. 초당 5개가 절대 넘어서는 안 된다면 token bucket이면 되나요?**  
먼저 ‘고정된 초’인지 ‘어느 시점에서든 직전 1초’인지 확인하겠습니다. 후자라면 이번 용량 5·충전율 5/s token bucket은 맞지 않습니다. 실험에서도 rolling 1초에 9개를 허용했습니다. Exact sliding log는 정의한 `(t-1000, t]`에서 최대 5개를 지켰습니다.

**Q. 여러 서버에 배포하면요?**  
서버마다 독립 한도를 두면 전역 한도가 아닙니다. 키 범위와 전역 쿼터를 정하고 공유 상태의 조회·만료·차감을 원자적으로 처리해야 합니다. 시계 기준, 저장소 장애 시 fail-open/fail-closed도 정해야 합니다. 이 코드는 그 동시성과 장애 동작을 검증하지 않았습니다.

**Q. 거절할 때 HTTP 응답은 어떻게 하나요?**  
요청 제한 초과에는 429를 고려하고 재시도 가능한 시점을 알려줄 수 있으면 `Retry-After`를 제공할 수 있습니다. 모든 정책에 고정된 대기 시간을 임의로 붙이지 않고 현재 상태에 맞춰 계산해야 합니다. 이 실험에는 HTTP 응답이나 대기 시간 계산을 구현하지 않았습니다.

## 흔한 오답

- “Fixed window도 모든 rolling 1초에서 5개를 보장합니다.” → 경계 전후 1ms 차이로 10개를 허용했습니다.
- “Token bucket이면 burst 없이 일정 간격으로 처리합니다.” → 용량만큼 burst를 허용하며 요청을 지연시키는 큐가 아닙니다.
- “Token bucket이 19개를 허용했으니 더 빠릅니다.” → 정책에 따른 허용 수 차이입니다. 지연 시간은 재지 않았습니다.
- “Sliding window면 모두 정확합니다.” → 이 실험은 exact log입니다. Sliding counter 같은 근사는 별개입니다.

## 참고

- [AWS API Gateway: token bucket의 rate와 burst 제한](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html)
- [RFC 6585 §4: 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585.html#section-4)
