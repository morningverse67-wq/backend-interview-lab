# 백엔드 면접 질문, 직접 돌려보고 답하기

![백엔드 면접 질문을 측정한 숫자로 답하는 backend-interview-lab](assets/cover.png)

> **Backend interview questions answered with runnable experiments.**  
> 외운 답 대신 **직접 실행하고 측정한 숫자**로 답하는 백엔드 기술 면접 정리입니다.

"N+1이 뭔가요?", "인덱스는 왜 빠른가요?" 같은 백엔드 면접 단골 질문을
**30초 답변 → 직접 측정한 근거 → 꼬리 질문 → 흔한 오답** 순서로 정리합니다.
모든 수치는 [backend-lab](https://github.com/morningverse67-wq/backend-lab)의 코드로 직접 다시 돌려 볼 수 있습니다(값은 환경에 따라 다름).

## 이 저장소가 다른 점

- **근거가 있는 답변**: "빠르다", "느리다" 대신 직접 잰 숫자와 실행 계획을 붙입니다.
- **재현 가능**: 실험 코드는 Python 표준 라이브러리만 써서 `python3 main.py` 한 줄로 돌아갑니다.
- **꼬리 질문까지**: 면접에서 실제로 이어지기 쉬운 두 번째, 세 번째 질문을 함께 다룹니다.
- **트레이드오프**: 해결책이 오히려 손해인 경우도 같이 적습니다.

## 질문 목록

✅ 답변 + 실측 완료 &nbsp; 🔜 준비 중

### 데이터베이스 · SQL

| # | 질문 | 상태 | 핵심 근거 |
|---|---|:---:|---|
| DB-01 | [N+1 문제란 무엇이고 어떻게 해결하나요?](database/n-plus-one.md) | ✅ | 쿼리 501번 321ms → 1번 1.7ms |
| DB-02 | [인덱스는 왜 빠르고, 언제 소용없나요?](database/index.md) | ✅ | 선택적 조회 887배, 90% 조회 0.9배 |
| DB-03 | [복합 인덱스의 컬럼 순서는 왜 중요한가요?](database/composite-index.md) | ✅ | 같은 두 컬럼, 순서만 바꿔 505배 차이 (0.08ms vs 42.07ms) |
| DB-04 | [OFFSET 페이징은 왜 느려지고, 커서 페이징은 언제 쓰나요?](database/offset-vs-keyset-paging.md) | ✅ | 10,000페이지 100회 조회 합계 OFFSET 187.96ms vs 커서 0.77ms (244.7배) |
| DB-05 | [인덱스를 못 타는 쿼리에는 어떤 것이 있나요?](database/non-sargable-queries.md) | ✅ | 인덱스는 그대로, WHERE만 고쳐서 날짜 함수 41.3배, `LIKE '%-1234'` 876.4배 (100회 조회 합계) |
| DB-06 | [커버링 인덱스란 무엇인가요?](database/covering-index.md) | ✅ | 테이블을 안 읽으니 한 달 합계 10회 82.20ms → 4.95ms (16.6배), 인덱스 밖 컬럼 조회 시 1.0배 |
| DB-07 | [대량 INSERT를 빠르게 하는 방법은?](database/bulk-insert.md) | ✅ | 1만 건을 매번 COMMIT 1,948.84ms → 트랜잭션 하나 8.08ms (241.2배), 대신 1건 실패에 0건 남음 |
| DB-08 | [COUNT(*)는 왜 느려지고, 대안은 무엇인가요?](database/count-cost.md) | ✅ | 100만 건 중 59.9만 건 세기: 인덱스 없음 21.16ms, 인덱스 6.12ms, 카운터 0.0005ms. 대신 카운터 때문에 INSERT 2.90배 느려짐 |

### 트랜잭션 · 동시성

| # | 질문 | 상태 | 핵심 근거 |
|---|---|:---:|---|
| TX-01 | [Lost update는 언제 생기고 낙관적 락으로 어떻게 막나요?](transaction/optimistic-lock.md) | ✅ | 한 행에 +1 요청 1,600번: 락 없이 최종 값 200(1,400건 유실), 낙관적 락 1,600. 대신 재시도 5,995번에 2.09배 느려짐 |
| TX-02 | [낙관적 락과 비관적 락은 언제 각각 쓰나요?](transaction/optimistic-vs-pessimistic-lock.md) | ✅ | 한 행에 +1 요청 1,600번: 낙관적 락 재시도 5,624번·p99 174.32ms, 비관적 락 재시도 0번·p99 18.92ms. 전체 시간은 2,767.2ms vs 2,771.7ms로 같음 |
| TX-03 | [재고 차감 동시성 문제를 어떻게 해결하나요?](transaction/stock-decrement.md) | ✅ | 재고 100개에 주문 400건: 읽고-쓰기 400개 판매, 원자적 UPDATE 정확히 100개. 주문 800건은 락보다 2.17배 빠름 (634.9ms vs 1,379.7ms) |
| TX-04 | [트랜잭션 격리 수준별로 어떤 현상이 생기나요?](transaction/isolation-levels.md) | ✅ | A를 읽고 0.2ms 뒤 B를 읽는 합계 보고서 1,000건: 문장마다 최신 커밋을 읽으면 235건 틀림, 읽기 트랜잭션 하나면 0건. 대신 스냅샷을 열어 두면 WAL 3.0배 (3.93MB → 11.79MB) |
| TX-05 | [중복 결제 요청은 어떻게 막나요? (멱등성 키)](transaction/idempotency-key.md) | ✅ | 같은 결제 요청 3번씩 주문 200건: 키를 먼저 조회하면 DB 행 200건인데 PG 결제 599번, 결제 전에 키부터 저장하면 200번. 시도마다 새 키면 400번 |
| TX-06 | [데드락은 왜 생기고 어떻게 피하나요?](transaction/deadlock.md) | ✅ | 계좌 2개 사이 양방향 이체 200건: 순서 없이 잠그면 데드락 250번(521.0ms), 작은 id부터 잠그면 0번(345.1ms). 감지 없이 30ms 타임아웃만 쓰면 6,658.9ms |

### 캐시

| # | 질문 | 상태 | 핵심 근거 |
|---|---|:---:|---|
| CA-01 | [캐시를 쓰면 얼마나 빨라지나요?](cache/lru-cache.md) | ✅ | 상품 페이지 2만 건, 같은 LRU 1,000개: 인기 상품에 몰리면 209.6ms → 77.5ms (히트 67.1%, 2.70배), 고르게 흩어지면 1.07배 (히트 9.7%). 무효화 없이 쓰기 2%면 읽기 36.3%가 옛 값 |
| CA-02 | [Cache stampede란 무엇이고 어떻게 막나요?](cache/cache-stampede.md) | ✅ | 캐시 만료 순간 같은 키로 요청 32개: 각자 쿼리하면 32번에 140.6ms, single-flight로 합치면 1번에 16.6ms (8.5배). 키 32개로 흩어지면 0.91배로 이득 없음 |
| CA-03 | [TTL에 jitter를 넣는 이유는?](cache/ttl-jitter.md) | ✅ | 키 1,000개를 같은 TTL 60초로 채우면 가장 바쁜 1초에 DB 조회 592번, 0~20% jitter면 96번 (±20%는 52번). 전체 조회 수는 거의 같음 |
| CA-04 | [Cache-aside, write-through, write-behind 차이는?](cache/write-policies.md) | ✅ | 쓰기 100번: 읽기 미스 cache-aside 100번 → write-through 0번, 키별 병합 write-behind는 UPDATE 50번. 대신 95번째 응답 후 캐시 유실 시 최신 값 없는 DB 키 10개 |

### API · 서버

| # | 질문 | 상태 | 핵심 근거 |
|---|---|:---:|---|
| API-01 | [Rate limiter 알고리즘별 차이는?](api/rate-limiter.md) | ✅ | 경계 요청 10개: fixed window 허용 10개, exact log·token bucket 5개. 800ms 재충전 burst에서 token bucket은 9개 허용 |
| API-02 | [재시도할 때 지수 백오프와 jitter를 쓰는 이유는?](api/retry-backoff-jitter.md) | ✅ | 1초 장애 뒤 클라이언트 100개: 지수 백오프만 쓰면 복구 후 거절 450번·마지막 성공 28,700ms, full jitter면 9번·4,048ms |
| API-03 | [Circuit breaker는 어떻게 동작하나요?](api/circuit-breaker.md) | ✅ | 초당 100개 요청, 1초 타임아웃: 장애 중 타임아웃 호출 breaker 없음 550번 → 109~118번. 대신 복구 후 거절 OPEN 500ms 75개, 5000ms 330개 |
| API-04 | [커넥션 풀 크기는 어떻게 정하나요?](api/connection-pool-size.md) | ✅ | 4코어 DB, 초당 800개: 풀 2개 포기 2,478개, 4개부터 0개. 풀 64개는 p99 25.80 → 40.86ms로 대기 장소만 이동. 커넥션을 20ms 더 쥐면 16개도 805개 포기 |
| API-05 | [I/O 작업은 동기, 스레드 풀, asyncio 중 무엇으로 처리하나요?](api/sync-threadpool-asyncio.md) | ✅ | 50ms 서버에 호출 100번: 순차 5,281.5ms, 스레드 100개 60.5ms, gather 65.3ms. `async def` 안 블로킹 호출은 5,254.4ms. CPU 반복문을 붙이면 스레드 100개도 579.2ms(CPU 합 592.5ms) |
| API-06 | [외부 API 호출에 타임아웃이 꼭 필요한 이유는?](api/external-call-timeout.md) | ✅ | 워커 8개, 외부 API 2초 멈춤: 타임아웃 없으면 `/health`도 최대 6,437.2ms, 읽기 타임아웃 200ms면 0.3ms(주문 504). 연결 타임아웃만은 6,439.1ms, 1바이트씩 흘리는 응답엔 읽기 타임아웃도 6,564.7ms |

### 데이터 처리

| # | 질문 | 상태 | 핵심 근거 |
|---|---|:---:|---|
| DP-01 | [대용량 CSV 파일을 메모리 걱정 없이 처리하려면 어떻게 하나요?](data/csv-streaming.md) | ✅ | 40만 행 23.4MB CSV: 한 번에 읽으면 메모리 최대치 +187.2MB(DictReader +245.5MB), 스트리밍 +0.1MB에 시간도 201.2 → 152.8ms. 중앙값은 금액 열만 남겨 +24.8MB |
| DP-02 | [정렬 병합 조인과 해시 조인은 언제 쓰나요?](data/merge-vs-hash-join.md) | ✅ | 8만×2만 행 Python join 실험에서 해시 조인 52.2~57.0ms, 병합 조인 195.8ms(정렬 입력)·200.2ms(정렬 포함); DBMS 실행 계획으로 일반화하지 않음 |
| DP-03 | [JSON, pickle, 바이너리 같은 직렬화 형식은 어떻게 고르나요?](data/serialization-format.md) | ✅ | 주문 10만 건: JSON 9.01MB, struct 2.10MB(4.29배), 압축 뒤 1.50 vs 1.21MB(1.24배). 필드를 추가하면 옛 struct 코드가 21건을 에러 없이 25건으로 잘못 읽음 |
| DP-04 | [Bloom filter는 무엇이고 언제 쓰나요?](data/bloom-filter.md) | ✅ | 키 10만 개를 125KB(키당 10비트)에: 오탐률 0.84%(공식 0.82%), 90%가 없는 키인 조회 10만 건 중 저장소 조회 10,728건. 2배를 넣으면 13.88%, 비트를 지워 삭제하면 남은 키 486개가 '없음' |
| DP-05 | [Consistent hashing은 왜 쓰나요?](data/consistent-hashing.md) | ✅ | 키 10만 개, 노드 10→11대: `hash % N`은 90,890개(90.89%) 이동, 가상 노드 100개 해시 링은 9,266개(9.27%, 이상값 9.09%)가 전부 새 노드로. 가상 노드 1개면 최대 노드가 평균의 3.72배 |
| DP-06 | [DB 저장과 메시지 발행은 어떻게 같이 처리하나요?](data/transactional-outbox.md) | ✅ | 주문 1만 건, 단계 사이 죽기 2%: 커밋 후 발행 197건 유실, 발행 후 커밋 유령 317건, outbox는 둘 다 0이고 216건 중복을 소비자 이벤트 id로 걸러 9,905건 처리 |
| DP-07 | [같은 메시지가 두 번 오면 소비자는 어떻게 처리하나요?](data/idempotent-consumer.md) | ✅ | 이벤트 1만 건, 재전송 3%·반영 뒤 죽기 2%: 그냥 반영 548건, id 따로 기록 223건 초과 반영, inbox와 반영 한 트랜잭션은 0 |
| DP-08 | [처리할 수 없는 메시지는 어떻게 다루나요?](data/dead-letter-queue.md) | ✅ | 메시지 1만 건 중 poison 129건, 시도 12,000번: 제자리 무한 재시도는 정상 메시지 38건만 처리, 최대 3번 시도 뒤 DLQ는 정상 9,871건 모두 처리하고 DLQ에는 poison 129건만 |
| DP-09 | [없으면 넣고 있으면 고치는 작업은 어떻게 처리하나요?](database/upsert.md) | ✅ | 같은 새 키에 동시 요청 1,000쌍: 조회 후 쓰기는 중복 키 오류 1,000번·증가분 1,000 사라짐, ON CONFLICT는 0이고 10만 건 98ms 대 45ms |
| DP-10 | [삭제한 데이터를 남겨 두면 유니크 제약은 어떻게 거나요?](database/soft-delete-partial-index.md) | ✅ | 10만 행 중 9만 행 삭제: 전체 UNIQUE는 탈퇴 이메일 재가입 1,000번 모두 오류·757페이지, 부분 UNIQUE는 재가입 오류 0·중복 1,000번 모두 막고 74페이지 |
| DP-11 | [최신순 목록 API가 느릴 때 인덱스는 어떻게 거나요?](database/order-by-limit-index.md) | ✅ | 주문 20만 행, 회원 한 명의 최근 20건: `INDEX(user_id)`는 임시 정렬·8,295단계·192.8ms, `INDEX(user_id, created_at)`는 정렬 없음·133단계·4.5ms |
| DP-12 | [오래 열린 트랜잭션이 왜 문제인가요?](database/long-transaction-cleanup.md) | ✅ | 1만 행에 1,000건씩 20번 UPDATE: 긴 읽기 없음은 WAL 0 MB, 긴 읽기를 열어 두면 정리 20/20 막힘·WAL 17.65 MB, 닫으면 0 MB |
| DP-13 | [대량으로 DELETE했는데 디스크가 그대로예요. 왜 그런가요?](database/bulk-delete-disk-space.md) | ✅ | 로그 20만 행 중 18만 행 DELETE: 파일 27.70 MB 그대로·빈 페이지 6,373개, `VACUUM` 뒤 2.77 MB |
| DP-14 | [분산 락에 임대 시간을 두면 안전한가요?](transaction/distributed-lock-fencing-token.md) | ✅ | 임대 30 ms 락으로 1만 번 증가: 임대 락만 9,756(244 잃음), fencing token 확인 시 늦은 쓰기 216건 거절·재시도로 10,000 |
| DP-15 | [Saga에서 중간 단계가 실패하면 어떻게 되돌리나요?](transaction/saga-compensation.md) | ✅ | 주문 1만 건 중 1,148건 실패: 보상 없으면 1,181건 어긋남, 재시도는 중복 반영 61건, 멱등 키 재시도는 0건 |
| DP-16 | [쓰기 직후 읽었는데 옛 값이 나와요. 왜 그런가요?](database/read-replica-lag.md) | ✅ | 쓰기 1만 건 직후 읽기: 항상 replica는 1,873건(18.73%) 옛 값, 쓴 뒤 1초 primary는 0건·primary 읽기 20.0%, 반영 위치 확인은 0건·3.7% |
| DP-17 | [없는 키 조회가 계속 저장소까지 가면 어떻게 줄이나요?](cache/negative-caching.md) | ✅ | 없는 키 조회 3만 71건: 매번 저장소는 3만 71번, TTL 5초는 1만 8,257번(39.3% 절감)·틀린 "없음" 5건, TTL 30초는 9,784번(67.5%)·61건, 만들 때 지우면 0건 |
| DP-18 | [캐시 서버를 재시작하면 왜 느려지고, 어떻게 줄이나요?](cache/cache-warmup.md) | ✅ | 재시작 직후 10초: 워밍업 없으면 적중률 78.5%·저장소 2,146번, 인기 키 500개 미리 채우면 83.3%·1,674번(22.0% 감소), 2,000개는 84.0%·1,596번이지만 전체 조회는 17,341번 |
| DP-19 | [로컬 캐시를 분산 캐시 앞에 두면 무엇이 좋아지고 무엇을 조심하나요?](cache/two-level-cache.md) | ✅ | 읽기 9만 7,985건: 분산 캐시만 평균 1.090 ms·옛 값 0건, 로컬 TTL 1초는 0.506 ms(53.6% 감소)·옛 값 17,488건(17.85%), TTL 10초는 0.291 ms·39,802건(40.62%) |

## 답변 형식

각 문서는 같은 순서를 따릅니다.

1. **30초 답변**: 면접에서 바로 말할 수 있는 길이
2. **조금 더 깊게**: 왜 그런지
3. **직접 측정한 근거**: 실험 조건, 결과 표, 재현 코드 링크
4. **꼬리 질문**: 이어서 나올 만한 질문과 짧은 답
5. **흔한 오답**: 이렇게 답하면 감점되기 쉬운 표현

## 관련 링크

- 실험 코드: [backend-lab](https://github.com/morningverse67-wq/backend-lab)
- 블로그: [velog.io/@morningverse](https://velog.io/@morningverse)

질문 제안이나 틀린 내용은 [Issue](https://github.com/morningverse67-wq/backend-interview-lab/issues)로 알려주세요.  
도움이 됐다면 ⭐ Star로 응원해 주세요.

<sub>Keywords: 백엔드 면접, 기술 면접, 신입 백엔드, backend interview, interview questions, database, SQL, N+1, index, transaction, concurrency, cache</sub>
