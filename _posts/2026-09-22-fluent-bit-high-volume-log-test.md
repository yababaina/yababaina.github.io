---
title: "Fluent Bit 로그 파이프라인 대용량 유입 테스트: 6.3K logs/s를 Drop 없이"
date: 2026-09-22 14:00:00 +0900
categories: [Logging]
tags: [Fluent Bit, Cisco TRex, Prometheus, Grafana, C/C++, Performance Test]
excerpt: "레거시 로깅 시스템을 Fluent Bit 기반 파이프라인으로 옮긴 뒤, Cisco TRex로 약 930Mb/s 트래픽을 흘려 장비 사양과 DPI, IDS/IPS 사용 여부별로 로그 처리량과 지연을 측정한 결과를 정리합니다."
---

네트워크 보안 장비의 레거시 로깅 시스템을 **Fluent Bit 기반 로그 파이프라인**으로 옮겼습니다. 이 글에서는 옮긴 파이프라인이 대용량 트래픽 상황에서도 로그를 잃지 않고 처리하는지 검증한 테스트를 정리합니다.

**결과 요약**

- 최대 **6.3K logs/s** 유입을 **Drop, Error 없이** 기록
- 동일 트래픽 조건에서 **DB 기록 건수가 레거시 시스템 대비 90% 이상 증가**
- 과부하 시 지연이 늘어난 원인은 Fluent Bit 내부가 아니라 **다른 프로세스의 CPU 점유**였고, 부하가 끝나면 1분 안에 정상으로 회복

---

# 파이프라인 구성

```
 로그 발생 프로세스 ──(UDS)────┐
                               ├──> [Input Plugin] ──> Fluent Bit ──> [Output Plugin] ──> PostgreSQL
 커널 모듈 ──────(Netlink)─────┘
                                             │
                                             └── metrics ──> Prometheus ──> Grafana
```

- **Input Plugin**: UDS(Unix Domain Socket), Netlink 기반 플러그인을 직접 구현해 사용자 공간 프로세스와 커널 양쪽의 로그를 수집
- **Output Plugin**: PostgreSQL에 기록하는 플러그인을 직접 구현
- **모니터링**: Fluent Bit metrics와 시스템 자원 사용량을 Prometheus로 수집하고 Grafana로 확인

---

# 테스트 개요

## 트래픽 조건

[Cisco TRex](https://trex-tgn.cisco.com/)로 트래픽을 만들어 장비에 흘렸습니다.

| 항목 | 값 |
| --- | --- |
| 테스트 시간 | 7분 |
| 평균 bps | 925 ~ 930 Mb/s |
| 평균 pps | 116 kp/s |
| 평균 cps | 3.33K c/s |
| 트래픽 방향 | `16.0.0.0/8` → `48.0.0.0/8` |

DPI, IDS/IPS 로그가 위 트래픽에 반응해 생성되도록 설정했습니다.

## 테스트 방법

- 사양이 다른 장비 2종에서 **DPI 사용 여부 × IDS/IPS 모드** 조합별로 측정했습니다.
- IPS를 켠 케이스는 `nf_queue` full로 트래픽 자체가 drop되어, 트래픽 양을 줄여 테스트했습니다.
- 측정 지표
    - **logs/s**: 초당 처리한 로그 수
    - **Chunk size**: Fluent Bit 내부에 쌓인 chunk 크기 (처리가 밀리면 커짐)
    - **latency**: 로그가 Fluent Bit에 들어온 뒤 output으로 flush될 때까지의 시간 (Fluent Bit metrics 기준)

---

# 테스트 결과

- 모든 케이스에서 **로그 drop, error 0건**
- 최대 유입량은 **약 6.3K logs/s**였고, 이때도 **P95 latency 1초 미만**
- 대부분의 케이스에서 P95 latency는 수 초 이내
- 일부 IDS 케이스에서만 과부하 시 latency가 최대 수십 초까지 늘어남

## latency가 크게 튄 케이스

IDS를 켠 일부 케이스에서 latency가 크게 늘었습니다. 확인해 보니 원인은 Fluent Bit가 아니었습니다.

- 트래픽 과부하로 **DPI 엔진과 IDS 엔진(Suricata)의 CPU 사용률이 올라가면서** Fluent Bit가 CPU를 충분히 받지 못해 latency가 상승했습니다.
- latency만 높았을 뿐, **로그 drop과 error는 0건**이었습니다.
- 테스트가 끝나 부하가 사라지면 **1분 안에** 정상 latency로 회복되었습니다.

---

# 정리 및 튜닝 포인트

## flush 주기는 장비 사양에 맞춰 조절해야 한다

| flush 주기 | latency | 대용량 유입 시 chunk 누적 | CPU 부담 |
| --- | --- | --- | --- |
| 짧게 | 낮음 | 적게 쌓임 | 높음 |
| 길게 | 높음 | 많이 쌓임 | 낮음 |

- 고사양 장비: flush를 0.5초로 설정해 latency를 최대한 줄였습니다.
- 저사양 장비: flush 0.5초에서는 테스트 중간에 Fluent Bit가 잠깐씩 멈추는 현상이 있었습니다. 1초로 늘려 해결했습니다.

## 대용량 로그는 input 플러그인 버퍼를 늘려야 한다

- 6K/s 이상 유입 시, 드물게 input 플러그인의 ring buffer가 순간적으로 가득 차거나, CPU 부하로 input 스레드가 메인 스레드로 로그를 넘기지 못하는 경우가 있었습니다.
- 관측된 실패는 테스트당 10건 미만이었고, **Fluent Bit가 재시도해 모두 에러나 drop 없이 전달**되었습니다.
- 그래도 대용량 유입 가능성이 있는 로그는 input 플러그인 내부 버퍼를 여유 있게 잡아 두는 것이 안전합니다.
