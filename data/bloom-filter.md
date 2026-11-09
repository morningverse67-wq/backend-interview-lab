# DP-04. Bloom filter는 무엇이고 언제 쓰나요?

## 30초 답변

Bloom filter는 비트 배열과 해시 몇 개로 "확실히 없음" 또는 "있을 수도 있음"만 답하는 구조입니다. 없는 키를 묻는 조회가 많을 때 저장소 앞에 두면, 확실히 없는 키는 저장소까지 가지 않게 거를 수 있습니다. 직접 재 보니 키 10만 개를 125KB(키당 10비트)에 넣었을 때 오탐률이 0.84%였고, 90%가 없는 키인 조회 10만 건 중 저장소까지 간 조회가 10,728건이었습니다. 대신 예상 키 수를 넘기면 오탐률이 빠르게 오르고, 일반 Bloom filter는 키를 지울 수 없다는 점을 같이 고려하겠습니다.

## 조금 더 깊게

```python
def positions(key):                       # 해시 k개 위치
    d = hashlib.blake2b(key.encode(), digest_size=16).digest()
    h1, h2 = int.from_bytes(d[:8], "little"), int.from_bytes(d[8:], "little") | 1
    return [(h1 + i * h2) % m for i in range(k)]

# add:      k개 위치의 비트를 모두 1로
# contains: k개 중 하나라도 0이면 "확실히 없음", 모두 1이면 "있을 수도 있음"
```

- 넣은 키는 비트가 반드시 켜져 있으니 **있는 키를 없다고 하는 일(false negative)은 없습니다.** 다른 키들이 우연히 같은 비트를 다 켜 두면 없는 키를 있다고 할 수 있는데, 이것이 오탐(false positive)입니다.
- 크기는 키 길이와 무관하게 키당 비트 수(m/n)로 정해지고, 해시 개수는 `k ≈ (m/n) × ln 2`일 때 오탐률이 가장 낮습니다. 오탐률은 `(1 - e^(-kn/m))^k`로 미리 계산할 수 있습니다.
- 쓰는 곳: 캐시에 없는 키가 계속 DB로 가는 cache penetration 막기, 아이디 중복 확인 앞단, LSM 트리 기반 저장소(RocksDB, Cassandra 등)가 디스크 파일을 읽기 전에 키가 없을 파일을 건너뛰는 용도.
- 주의: "있을 수도 있음"이면 반드시 실제 저장소로 확인해야 합니다. 필터의 "있음"을 결과로 쓰면 안 됩니다.

## 직접 측정한 근거

사용자 ID 10만 개를 넣고, 조회 10만 건(있는 키 10%, 없는 키 90%)을 보냈습니다. 오탐률은 넣지 않은 키 5만 개로 쟀습니다.

| bits/key | k | 필터 크기 | 오탐률 측정 | 오탐률 공식 | 저장소 조회 | false negative |
|---:|---:|---:|---:|---:|---:|---:|
| 필터 없음 | - | - | - | - | 100,000 | - |
| 4 | 3 | 50.0KB | 14.77% | 14.69% | 23,259 | 0 |
| 8 | 6 | 100.0KB | 2.26% | 2.16% | 12,005 | 0 |
| 10 | 7 | 125.0KB | 0.84% | 0.82% | 10,728 | 0 |
| 16 | 11 | 200.0KB | 0.03% | 0.05% | 10,045 | 0 |

| 10 bits/key, 10만 건 기준 크기에 넣은 키 | 100,000 | 200,000 | 300,000 | 400,000 |
|---|---:|---:|---:|---:|
| 오탐률 측정 | 0.84% | 13.88% | 40.18% | 64.54% |

- 같은 키를 담은 Python `set`의 해시 테이블은 4.19MB(문자열 제외)로, 10 bits/key 필터의 약 34배였습니다.
- 키 100개를 지우려고 그 키들의 비트를 0으로 되돌리자, 남은 99,900개 중 486개가 "없음"으로 나왔습니다.
- 재현 코드: [backend-lab/029-bloom-filter](https://github.com/morningverse67-wq/backend-lab/tree/main/labs/029-bloom-filter)

<sub>Python 3.13.12, macOS arm64. KB·MB는 1,000 / 1,000,000바이트. 오탐률·조회 수는 시드 고정. 환경에 따라 수치는 달라질 수 있음.</sub>

## 꼬리 질문

**Q. 오탐이 있으면 문제가 되지 않나요?**  
오탐이면 저장소에 한 번 더 물어보는 것뿐이라 결과는 틀리지 않습니다. 비용이 조금 늘 뿐입니다. 이 실험에서 10 bits/key면 없는 키 90,000건 중 오탐으로 저장소에 간 것은 728건이었습니다. 반대로 false negative가 생기면 있는 데이터를 없다고 하게 되니 그쪽이 훨씬 위험합니다.

**Q. 키를 지워야 하면요?**  
일반 Bloom filter는 비트 하나를 여러 키가 같이 쓰기 때문에 지울 수 없습니다. 실험에서 비트를 꺼서 지우자 상관없는 키 486개가 없다고 나왔습니다. 삭제가 필요하면 비트 대신 카운터를 두는 counting Bloom filter를 쓰거나, 주기적으로 필터를 새로 만들겠습니다.

**Q. 그냥 set이나 해시맵에 넣으면 안 되나요?**  
키가 메모리에 다 들어가면 그게 더 단순하고 정확합니다. 이 실험의 Python 구현에서도 조회 자체는 `set`이 수십 배 빨랐습니다. Bloom filter는 키가 너무 많아 다 들고 있기 어렵거나, 뒤의 저장소 조회가 디스크·네트워크라서 필터 계산보다 훨씬 비쌀 때 의미가 있습니다.

## 흔한 오답

- "Bloom filter가 있다고 하면 있는 겁니다" → "있을 수도 있음"일 뿐입니다. 10 bits/key에서도 없는 키의 0.84%를 있다고 했습니다.
- "한 번 크기를 정하면 계속 쓸 수 있습니다" → 예상보다 키가 2배가 되자 오탐률이 0.84%에서 13.88%로 올랐습니다. 에러 없이 저장소 조회만 늘어납니다.

## 참고

- [Wikipedia: Bloom filter](https://en.wikipedia.org/wiki/Bloom_filter)
- [Kirsch, Mitzenmacher, "Less Hashing, Same Performance: Building a Better Bloom Filter"](https://www.eecs.harvard.edu/~michaelm/postscripts/rsa2008.pdf)
- [Apache Cassandra 문서: Bloom Filters](https://cassandra.apache.org/doc/latest/cassandra/managing/operating/bloom_filters.html)
- [Python 문서: hashlib](https://docs.python.org/3/library/hashlib.html)
