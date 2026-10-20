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
| TX-02 | 낙관적 락과 비관적 락은 언제 각각 쓰나요? | 🔜 | |
| TX-03 | 재고 차감 동시성 문제를 어떻게 해결하나요? | 🔜 | |
| TX-04 | 트랜잭션 격리 수준별로 어떤 현상이 생기나요? | 🔜 | |
| TX-05 | 중복 결제 요청은 어떻게 막나요? (멱등성 키) | 🔜 | |
| TX-06 | 데드락은 왜 생기고 어떻게 피하나요? | 🔜 | |

### 캐시

| # | 질문 | 상태 | 핵심 근거 |
|---|---|:---:|---|
| CA-01 | 캐시를 쓰면 얼마나 빨라지나요? | 🔜 | |
| CA-02 | Cache stampede란 무엇이고 어떻게 막나요? | 🔜 | |
| CA-03 | TTL에 jitter를 넣는 이유는? | 🔜 | |
| CA-04 | Cache-aside, write-through, write-behind 차이는? | 🔜 | |

### API · 서버

| # | 질문 | 상태 | 핵심 근거 |
|---|---|:---:|---|
| API-01 | Rate limiter 알고리즘별 차이는? | 🔜 | |
| API-02 | 재시도할 때 지수 백오프와 jitter를 쓰는 이유는? | 🔜 | |
| API-03 | Circuit breaker는 어떻게 동작하나요? | 🔜 | |
| API-04 | 커넥션 풀 크기는 어떻게 정하나요? | 🔜 | |
| API-05 | 외부 API 호출에 타임아웃이 꼭 필요한 이유는? | 🔜 | |

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
