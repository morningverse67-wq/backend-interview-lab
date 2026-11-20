# DP-16. 쓰기 직후 읽었는데 옛 값이 나와요. 왜 그런가요?

## 30초 답변

읽기를 replica로 나누면 복제가 끝나기 전에 읽는 요청이 옛 값을 받을 수 있습니다. 직접 재 보니 쓰기 1만 건 바로 뒤 읽기에서 replica만 읽으면 1,873건(18.73%)이 옛 값이었습니다. 쓴 뒤 1초는 primary를 읽으면 0건이었지만 전체 읽기의 20.0%가 primary로 갔고, replica가 그 쓰기를 반영했는지 확인해 돌리면 0건에 primary 읽기는 3.7%였습니다.

## 조금 더 깊게

- 복제는 비동기라 쓰기와 replica 반영 사이에 지연이 있습니다. 평균이 짧아도 가끔 튀는 지연이 어긋남을 만듭니다.
- 방금 쓴 사용자의 읽기만 primary로 보내는 방식(read-your-writes)이 흔한 해법입니다.
- 시간 창 방식은 구현이 쉽지만 지연이 창보다 길면 막지 못하고, 반영 위치 확인은 정확한 대신 구현이 더 듭니다.

## 직접 측정한 근거

| 방법 | 옛 값 읽기 | 전체 읽기 중 primary |
|---|---:|---:|
| 항상 replica | 1,873 (18.73%) | 0.0% |
| 쓴 뒤 1초는 primary | 0 | 20.0% |
| 반영 위치 확인 후 replica | 0 | 3.7% |
| 항상 primary | 0 | 100.0% |

- 재현 코드: [backend-lab/041-read-replica-lag](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/041-read-replica-lag)

<sub>Python 3.13.12, macOS arm64, 시드 41 시뮬레이션. 여러 번 돌려도 숫자가 같았습니다.</sub>

## 꼬리 질문

**Q. 복제 지연은 어떻게 확인하나요?**  
DB가 주는 복제 지연 지표를 보고, 평균보다 튀는 값(상위 백분위)을 함께 봅니다.

**Q. 모든 읽기를 primary로 보내면 안 되나요?**  
옛 값은 사라지지만 실험에서 primary 읽기가 100%가 돼 replica를 둔 이유가 없어집니다.

## 흔한 오답

- "replica 지연은 평균이 짧으니 괜찮습니다" → 튀는 지연이 쓰기 직후 읽기 간격보다 길면 그 요청이 옛 값을 받습니다.

## 참고

- [Amazon Aurora: Replica lag](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Replication.html)
- [PostgreSQL: Hot standby](https://www.postgresql.org/docs/current/hot-standby.html)
