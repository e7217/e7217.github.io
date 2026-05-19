---
title: "EDG 데이터 플레인의 신뢰성 경계 정리"
date: 2026-05-19 01:00:00 +0900
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

EDG는 산업용 설비 데이터를 엣지에서 수집하고 검증한 뒤 저장 계층으로 흘려보내는 게이트웨이다. 이런 시스템에서 "신뢰성 있는 수집"이라는 표현은 조심해서 써야 한다. 센서 값 하나가 여러 홉을 지나기 때문에, 어느 지점부터 내구성이 보장되는지 명확히 하지 않으면 운영자와 어댑터 작성자가 서로 다른 기대를 갖게 된다.

이번 작업의 핵심은 EDG의 데이터 플레인에서 **at-least-once delivery가 어디서 시작되는지**를 명시하는 것이었다.

## 배경

EDG의 데이터 흐름은 대략 이렇게 나뉜다.

```text
adapter
  -> platform.data.asset
  -> EDG Core
  -> platform.data.validated
  -> Telegraf
  -> VictoriaMetrics
```

처음에는 전체 흐름을 통째로 "안정적으로 저장된다"고 설명하기 쉬웠다. 하지만 실제 구현을 보면 어댑터에서 코어로 들어오는 홉은 일반 NATS publish이고, 코어에서 검증된 데이터를 내보내는 홉은 NATS JetStream publish ack를 기다린다. 두 구간은 같은 신뢰성 모델이 아니다.

그래서 ADR 0001에서는 EDG의 신뢰성 경계를 다음처럼 고정했다.

> EDG의 at-least-once delivery는 코어가 `platform.data.validated`에 JetStream publish ack를 받은 뒤부터 시작한다.

이 문장이 중요하다. 어댑터가 `platform.data.asset`에 publish했다고 해서 그 데이터가 내구적으로 저장됐다고 볼 수는 없다. 강한 보장이 필요한 어댑터는 그 앞단에서 재시도나 로컬 버퍼링을 직접 가져야 한다.

## JetStream 기본 정책

EDG의 기본 스트림은 `PLATFORM_DATA`다. subject는 `platform.data.>`를 잡고, 파일 스토리지 기반으로 7일 또는 1GiB까지 보관한다.

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

## Dead-letter와 expvar 카운터

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

## 어댑터 작성자에게 주는 의미

이 결정 이후 어댑터 작성자의 책임이 더 분명해졌다.

- 일반 NATS publish 성공은 내구성 보장이 아니다.
- 어댑터가 설비 값을 반드시 잃지 않아야 한다면 로컬 큐나 재시도 전략이 필요하다.
- 코어 이후 구간은 JetStream ack를 기준으로 관측하고 복구한다.
- Telegraf는 durable consumer로 검증 스트림을 읽고, downstream write 이후 ack한다.

즉, EDG가 모든 구간을 마법처럼 보장하는 것이 아니라, 보장하는 구간과 보장하지 않는 구간을 명확히 나눈다.

## 배운 점

산업용 데이터 수집 시스템에서는 기능보다 경계가 더 중요할 때가 있다. "데이터를 받는다", "저장한다", "보장한다" 같은 표현은 구현 단계에서 구체적인 subject, ack, stream policy, failure counter로 내려와야 한다.

이번 ADR의 가치는 JetStream을 붙인 것 자체보다, EDG가 책임지는 구간을 운영자와 어댑터 작성자가 같은 언어로 이해하게 만든 데 있다.

관련 저장소: [github.com/e7217/edg](https://github.com/e7217/edg)
