---
title: "산업용 데이터 수집에서 저장 보장의 경계 정하기"
date: 2026-05-19 01:00:00 +0900
excerpt: "설비 데이터 수집 파이프라인에서 publish 성공과 저장 보장은 같은 말이 아니다. 어느 지점부터 내구성을 보장할지 정리한 기록."
categories:
  - smart-factory
  - engineering
tags:
  - edg
  - nats
  - jetstream
  - reliability
  - dead-letter
  - victoria-metrics
---

설비 데이터 수집에서 가장 위험한 착각은 "publish가 성공했으니 저장됐다"고 믿는 것이다. 센서 값 하나가 여러 홉을 지나기 때문에, 어느 지점부터 내구성이 보장되는지 명확히 하지 않으면 운영자와 어댑터 작성자가 서로 다른 기대를 갖게 된다.

이 글은 개인 프로젝트인 **EDG Platform**에서 데이터 플레인의 신뢰성 경계를 정리한 기록이다. 핵심은 **at-least-once delivery가 어디서 시작되는지**를 코드, 설정, 문서에 같은 언어로 남기는 것이었다.

## 요약

- 문제: 어댑터 publish 성공과 저장 계층의 내구성 보장이 섞여 있었다.
- 결정: `platform.data.validated`에 대한 JetStream publish ack 이후부터 at-least-once delivery로 정의했다.
- 구현: 검증 스트림, dead-letter subject, expvar counter, ADR 문서를 함께 추가했다.
- 남은 과제: 어댑터 앞단의 end-to-end ack나 로컬 버퍼링은 별도 설계가 필요하다.

## 프로젝트 맥락

EDG의 데이터 흐름은 대략 이렇게 나뉜다.

```text
adapter
  -> platform.data.asset
  -> EDG Core
  -> platform.data.validated
  -> Telegraf
  -> VictoriaMetrics
```

EDG는 산업용 설비 데이터를 엣지에서 수집하고 검증한 뒤 저장 계층으로 흘려보내는 게이트웨이다. 처음에는 전체 흐름을 통째로 "안정적으로 저장된다"고 설명하기 쉬웠다.

하지만 실제 구현을 보면 어댑터에서 코어로 들어오는 홉은 일반 NATS publish이고, 코어에서 검증된 데이터를 내보내는 홉은 NATS JetStream publish ack를 기다린다. 두 구간은 같은 신뢰성 모델이 아니다.

## 문제

"신뢰성 있는 수집"이라는 말은 너무 넓다. 구체적으로는 다음 질문에 답해야 했다.

- 어댑터가 NATS에 publish하면 저장된 것으로 볼 수 있는가?
- 코어가 검증한 payload는 어디에 내구적으로 남는가?
- 저장 실패를 운영자가 어떻게 감지하는가?
- 어댑터 작성자는 어느 지점까지 직접 재시도해야 하는가?

이 질문에 답하지 않으면 운영자는 EDG가 전체 구간을 보장한다고 오해할 수 있고, 어댑터 작성자는 로컬 버퍼링을 생략할 수 있다.

## 선택지

| 선택지 | 장점 | 단점 | 판단 |
|---|---|---|---|
| 전체 구간을 best-effort로 둔다 | 구현이 단순하다 | "신뢰성"을 설명할 수 없다 | 제외 |
| 어댑터부터 저장소까지 end-to-end ack를 만든다 | 의미가 가장 명확하다 | request/reply, 로컬 큐, 어댑터 변경이 커진다 | 이후 과제 |
| 코어 이후를 JetStream 기준으로 보장한다 | 현재 구조에서 명확한 경계를 만들 수 있다 | 어댑터 앞단은 여전히 별도 책임이다 | 선택 |

이번 단계에서는 세 번째를 선택했다. EDG가 이미 코어 이후 validated stream을 갖고 있었고, JetStream publish ack는 운영자가 이해할 수 있는 명확한 경계였다.

## 결정

ADR 0001에서는 EDG의 신뢰성 경계를 다음처럼 고정했다.

> EDG의 at-least-once delivery는 코어가 `platform.data.validated`에 JetStream publish ack를 받은 뒤부터 시작한다.

이 문장이 중요하다. 어댑터가 `platform.data.asset`에 publish했다고 해서 그 데이터가 내구적으로 저장됐다고 볼 수는 없다. 강한 보장이 필요한 어댑터는 그 앞단에서 재시도나 로컬 버퍼링을 직접 가져야 한다.

## JetStream 기본 정책

기본 스트림은 `PLATFORM_DATA`다. subject는 `platform.data.>`를 잡고, 파일 스토리지 기반으로 7일 또는 1GiB까지 보관한다.

```text
Stream    : PLATFORM_DATA
Subjects  : platform.data.>
Storage   : file
Retention : limits
Max age   : 168h
Max bytes : 1 GiB
Replicas  : 1
Discard   : old
```

단일 노드 엣지 게이트웨이를 먼저 목표로 했기 때문에 replica는 1이다. 멀티 노드 복제는 배포, 스토리지, 업그레이드 모델까지 바꾸는 문제라 이번 결정에서는 제외했다.

`DiscardOld`도 의도적인 선택이다. 스트림이 꽉 찼을 때 새 데이터를 거부하는 대신 오래된 데이터를 밀어낸다. 이 정책은 운영자가 저장소 압박을 모니터링해야 한다는 책임을 만든다. 그래서 설정값과 함께 관측 지표가 필요했다.

## 실패를 드러내는 장치

검증된 데이터를 JetStream으로 publish할 때 실패할 수 있다. 이때 단순히 로그만 남기면 운영자가 손실 가능성을 늦게 발견한다. 그래서 실패한 publish는 `platform.data.deadletter`로 감싼다.

```json
{
  "original_subject": "platform.data.asset",
  "target_subject": "platform.data.validated",
  "error": "...",
  "payload": { "...": "..." },
  "timestamp": "..."
}
```

그리고 코어는 expvar 카운터를 노출한다.

```text
edg_core_jetstream_publish_failures
edg_core_jetstream_dead_letters
edg_core_jetstream_dead_letter_failures
```

이 카운터들은 "데이터가 잘 저장되고 있다"보다 "어디서 압력이 생기는지"를 보기 위한 장치다. 특히 dead-letter 자체도 publish에 실패할 수 있기 때문에 별도 실패 카운터를 둔 점이 중요했다.

## 책임 경계

이 결정 이후 어댑터 작성자의 책임이 더 분명해졌다.

- 일반 NATS publish 성공은 내구성 보장이 아니다.
- 어댑터가 설비 값을 반드시 잃지 않아야 한다면 로컬 큐나 재시도 전략이 필요하다.
- 코어 이후 구간은 JetStream ack를 기준으로 관측하고 복구한다.
- Telegraf는 durable consumer로 검증 스트림을 읽고, downstream write 이후 ack한다.

즉, EDG가 모든 구간을 마법처럼 보장하는 것이 아니라, 보장하는 구간과 보장하지 않는 구간을 명확히 나눈다.

## 검증

이 변경은 문서만으로 끝내지 않았다. 코어 테스트에는 다음 시나리오를 넣었다.

- JetStream consumer가 나중에 붙어도 backlog를 회수하는지
- 작은 `MaxBytes`에서 `DiscardOld` 정책이 의도대로 동작하는지
- 같은 자산이 동시에 auto-registration될 때 중복 문제가 없는지
- validated publish 실패 시 dead-letter subject로 envelope가 나가는지

이 테스트들은 신뢰성 경계가 말뿐인 문서가 아니라 실제 동작으로 유지되는지 확인하기 위한 회귀 테스트다.

## 남은 과제

이번 결정은 코어 이후 구간의 경계를 정한 것이다. 아직 남은 일도 분명하다.

- 어댑터에서 코어까지의 end-to-end acknowledgement
- 네트워크가 불안정한 설비망에서의 로컬 buffer/flush 전략
- JetStream storage pressure를 운영 화면이나 알림으로 드러내는 방식
- 멀티 노드 JetStream replication을 도입할 때의 배포 모델

## 배운 점

산업용 데이터 수집 시스템에서는 기능보다 경계가 더 중요할 때가 있다. "데이터를 받는다", "저장한다", "보장한다" 같은 표현은 구현 단계에서 구체적인 subject, ack, stream policy, failure counter로 내려와야 한다.

이번 ADR의 가치는 JetStream을 붙인 것 자체보다, EDG가 책임지는 구간을 운영자와 어댑터 작성자가 같은 언어로 이해하게 만든 데 있다.

관련 자료:

- 저장소: [github.com/e7217/edg](https://github.com/e7217/edg)
- ADR: [docs/adr/0001-data-plane-reliability.md](https://github.com/e7217/edg/blob/main/docs/adr/0001-data-plane-reliability.md)
- 구현: [internal/core/handler.go](https://github.com/e7217/edg/blob/main/internal/core/handler.go)
- 설정: [internal/core/config.go](https://github.com/e7217/edg/blob/main/internal/core/config.go)
