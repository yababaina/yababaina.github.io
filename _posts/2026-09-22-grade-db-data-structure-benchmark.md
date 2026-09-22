---
title: "등급 DB 조회 자료구조 벤치마크: vector vs hash table vs MPHF, 그리고 Bloom filter"
date: 2026-09-22 10:00:00 +0900
categories: [Optimization]
tags: [C++, MPHF, PTHash, Bloom Filter, Benchmark]
excerpt: "유해사이트 등급 DB를 메모리에 올려 실시간으로 조회하기 위해 vector, hash table(robin_hood), MPHF(PTHash)를 Bloom filter 적용 여부에 따라 비교한 벤치마크 결과를 정리합니다."
---

트래픽에서 추출한 Host가 유해사이트인지 실시간으로 판정하려면, 등급 DB를 메모리에 올려 두고 매 요청마다 빠르게 조회해야 합니다. 이때 어떤 자료구조를 쓸지 정하기 위해 후보 자료구조의 **메모리 사용량, 초기화 시간, 조회 시간**을 비교했습니다.

**결론 먼저**

- **Bloom filter**를 앞단에 두면 lookup miss가 자료구조와 상관없이 **약 145~155ns**로 수렴합니다. (MPHF 기준 340ns → 148ns)
- 메모리 제약이 있는 환경에서는 **RSS가 가장 작은 MPHF + Bloom filter** 조합이 적합합니다.

---

# 테스트 개요

유해사이트 등급 DB를 대상으로, 각 row의 **Host MD5(16B)와 등급(4B)**만 메모리에 적재한 뒤 조회 성능을 비교했습니다.

**비교 대상**

| 자료구조 | 구현 |
| --- | --- |
| vector | C++ STL `std::vector` |
| hash table | [robin_hood](https://github.com/martinus/robin-hood-hashing) |
| MPHF | [PTHash](https://github.com/jermp/pthash) |

각 자료구조를 **Bloom filter 적용 여부**에 따라 나눠 총 6개 케이스를 측정했습니다.

# 측정 기준

| 항목 | 지표 | 도구 | 조건 |
| --- | --- | --- | --- |
| 메모리 사용량 | VmRSS (MiB) | `/proc/self/status` | init 직후 측정. Bloom filter + 자료구조가 실제로 쓰는 RAM |
| 조회 속도 | lookup 1회당 평균 지연 (ns) | `std::chrono::steady_clock` | 샘플 key 1,000개를 순환하며 hit/miss 각 100,000회 반복 |
| 초기화 속도 | init 구간 (ms) | `std::chrono::steady_clock` | DB 파일을 읽어 자료구조를 만드는 데 걸린 시간 |

---

# 측정 결과

## 자료구조 단독 (Bloom filter 없음)

| | RSS (MiB) | init (ms) | lookup hit (ns) | lookup miss (ns) |
| --- | --- | --- | --- | --- |
| vector | 20.9 | 1,496 | 1,820 | 1,671 |
| hash | 45.8 | **347** | **194** | **122** |
| MPHF | **14.6** | 2,093 | 381 | 340 |

## 자료구조 + Bloom filter

| | RSS (MiB) | init (ms) | lookup hit (ns) | lookup miss (ns) |
| --- | --- | --- | --- | --- |
| vector | 22.4 | 1,739 | 1,987 | 155 |
| hash | 47.3 | **540** | **451** | **146** |
| MPHF | **16.1** | 2,276 | 641 | 148 |

- Bloom filter 상주 크기는 약 1.6MB입니다.

---

# 분석

## Bloom filter

- Bloom filter 없이 miss가 나면 자료구조의 조회 경로를 끝까지 타야 하므로, 자료구조에 따라 **122~1,671ns**로 편차가 큽니다.
- Bloom filter를 적용하면 DB에 없는 키 대부분이 Bloom filter에서 걸러져, lookup miss가 **약 145~155ns**로 수렴합니다.
- 대신 hit에서는 Bloom filter를 한 번 더 거치므로 지연이 늘어납니다.

실제 트래픽에서는 DB에 등록되지 않은 정상 Host가 대부분입니다. 즉 **miss가 압도적으로 많기 때문에**, hit 비용이 조금 늘더라도 Bloom filter를 1차 거름망으로 두는 것이 전체적으로 유리합니다.

## 자료구조

등급 DB는 갱신 빈도가 낮은 정적 데이터이므로, init 시간은 상대적으로 덜 중요합니다. 그래서 **메모리**와 **조회 속도**를 기준으로 판단했습니다.

- **MPHF**: RSS가 가장 작습니다. hit 지연은 hash보다 크지만, miss 비중이 높은 실제 트래픽에서는 Bloom filter 효과가 더 큽니다. 메모리 제약이 있는 임베디드 환경에 적합합니다.
- **hash table**: 조회가 가장 빠르지만 메모리를 약 3배 사용합니다. 메모리 여유가 있고 hit 지연을 최소화해야 할 때 대안이 될 수 있습니다.
- **vector**: 조회가 너무 느려 실사용에는 부적합합니다.

**선택: MPHF + Bloom filter**

---

# 남은 문제

MPHF를 선택하긴 했지만, 이 벤치마크에서 MPHF는 **init이 약 2.3초로 가장 느렸습니다.** 원본 DB를 런타임에 읽어 MPHF를 빌드하기 때문이고, 빌드 중에는 원본 DB 전체가 메모리에 올라가 순간적으로 메모리 사용량도 커집니다.

이 문제를 MPHF를 오프라인에서 미리 빌드한 바이너리 인덱스로 해결한 과정은 다음 글에 정리했습니다.

- [350MB 유해사이트 DB를 17MB 인덱스로: MPHF 기반 조회 모듈 설계]({% post_url 2026-09-22-harmful-site-db-index-optimization %})
