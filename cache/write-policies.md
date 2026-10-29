# CA-04. Cache-aside, write-through, write-behind 차이는?

## 30초 답변

Cache-aside는 읽기 미스 때 애플리케이션이 DB에서 캐시를 채우고, 쓰기 때는 DB 갱신 후 캐시를 지우는 식으로 관리합니다. Write-through는 쓰기 경로에서 DB와 캐시를 갱신한 뒤 응답하고, write-behind는 캐시에 먼저 쓰고 DB에는 나중에 반영합니다. 후자는 같은 키의 쓰기를 합칠 수 있지만, DB에 반영하기 전에 응답하므로 휘발성 대기열이 사라지면 응답한 값도 잃을 수 있습니다. 어떤 전략이든 DB와 캐시 사이의 부분 실패와 동시 읽기까지 별도로 설계해야 합니다.

## 조금 더 깊게

- **Cache-aside**: 이 실험은 DB 커밋 → 캐시 삭제 → 응답 순서입니다. 다시 읽을 때 미스가 나지만, 읽지 않는 값을 미리 캐시에 넣지 않습니다.
- **Write-through**: DB 커밋 → 캐시 갱신 → 응답으로 구현했습니다. 쓰기 직후 읽기에 유리하지만, 두 저장소가 하나의 원자적 트랜잭션이 되는 것은 아닙니다. DB 커밋 뒤 캐시 갱신이 실패하는 경우도 정해야 합니다.
- **Write-behind**: 캐시와 메모리 대기열 갱신 → 응답 → 나중에 DB 반영입니다. 여기서는 쓰기 20번마다 키별 마지막 값만 저장했습니다. 내구성 있는 대기열, 재처리, 순서와 중복 처리는 이 예제 밖의 설계입니다.

## 직접 측정한 근거

![정상 종료 시 DB UPDATE는 cache-aside 100번, write-through 100번, write-behind 50번. 95번째 응답 직후 캐시와 대기열을 버리면 DB에 최신 값이 없는 키는 각각 0개, 0개, 10개입니다.](../assets/ca-04-write-policies.svg)

키 10개를 순서대로 반복하며 쓰기 100번과 직후 읽기 100번을 실행했습니다. SQLite 메모리 DB와 Python dict로 동작 순서를 재현했고 초기 적재·검증 조회는 작업 수에서 뺐습니다.

| 정책 | 정상 종료 DB UPDATE | 읽기 미스 | 95번째 응답 후 캐시 유실 시 최신 값이 없는 DB 키 |
|---|---:|---:|---:|
| Cache-aside | 100 | 100 | 0 |
| Write-through | 100 | 0 | 0 |
| Write-behind, 키별 병합 | 50 | 0 | 10 |

장애 실험에서는 80번째 쓰기까지 반영한 뒤 15번 더 응답했습니다. DB UPDATE는 40번, 대기열에 남은 키는 10개였습니다. 미반영 응답 15번과 최신 값이 없는 키 10개는 서로 다른 지표입니다. DB가 살아 있고 캐시 메모리만 사라지는 상황을 모사했습니다. 실제 프로세스 종료·디스크 복구·지연 시간·처리량을 측정한 것은 아닙니다.

- 재현 코드와 전체 결과: [backend-lab/019-cache-write-policies](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/019-cache-write-policies)
- 실행: `python3 main.py` (Python 3 표준 라이브러리만 사용)

## 꼬리 질문

**Q. Write-behind면 DB 쓰기가 항상 줄어드나요?**  
아닙니다. 이번에는 20번 쓰는 동안 키마다 두 번 쓰고 마지막 값만 남겨 100번이 50번으로 줄었습니다. 병합하지 않으면 100번입니다. 매번 보존해야 하는 이벤트나 결제 이력을 마지막 값만 남기면 안 됩니다.

**Q. DB를 먼저 바꾸고 캐시를 지우면 일관성이 보장되나요?**  
아닙니다. 먼저 시작한 읽기가 옛 DB 값을 들고 있다가 쓰기와 캐시 삭제가 끝난 뒤 캐시를 채울 수 있습니다. 예제에서는 DB가 1인데 캐시는 0인 순서를 재현했습니다. 버전 검증이나 읽기 채우기와 쓰기 간 동기화가 필요하고, TTL만으로 경합을 없애지는 못합니다.

**Q. 결제처럼 응답한 값을 잃으면 안 된다면요?**  
단순한 휘발성 write-behind 예제는 쓰지 않겠습니다. DB 커밋 또는 내구성 있는 로그의 기록을 확인한 뒤 응답하고, 비동기 소비의 재시도·중복·순서를 설계해야 합니다. 이 실험은 그 내구성을 검증하지 않았습니다.

## 흔한 오답

- “Write-through면 항상 강한 일관성입니다.” → DB와 캐시 사이 부분 실패와 동시 읽기는 별개입니다.
- “Write-behind는 두 배 빠릅니다.” → 이번에 줄어든 것은 UPDATE 수입니다. 지연 시간은 재지 않았습니다.
- “캐시에서 최신 값이 읽히면 저장이 끝났습니다.” → write-behind는 캐시에서는 최신 값이어도 DB는 이전 값일 수 있습니다.

## 참고

- [Microsoft: Cache-Aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [AWS: Database caching strategies](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html)
