---
title: "EDG 어댑터 SDK와 Modbus TCP 예제 확장기"
date: 2026-05-19 01:20:00 +0900
categories:
  - smart-factory
  - engineering
tags:
  - edg
  - adapter
  - sdk
  - modbus
  - go
  - python
---

EDG에서 가장 오래 갈 인터페이스는 SDK가 아니라 wire contract라고 봤다. 어댑터가 결국 해야 하는 일은 정해진 NATS subject에 정해진 JSON payload를 publish하고, 필요한 경우 메타데이터 subject를 호출하는 것이다.

그렇다고 모든 어댑터 작성자가 매번 NATS 연결, 재시도, payload 모델, 관계 생성 코드를 직접 써야 하는 것은 아니다. 그래서 EDG는 **wire contract를 우선으로 두고, SDK는 편의 계층으로 제공하는 방향**을 택했다.

## 왜 SDK보다 subject 계약을 먼저 봤나

산업용 프로토콜은 언어 선택이 제각각이다. 어떤 현장은 Python 라이브러리가 편하고, 어떤 프로토콜은 Go나 C/C++ 바인딩이 더 낫다. 특정 SDK를 강제하면 어댑터 작성의 자유도가 줄어든다.

그래서 EDG의 기본 계약은 작게 유지했다.

```text
platform.data.asset
platform.data.validated
platform.meta.asset.*
platform.meta.relation.*
platform.meta.*.changed
```

이 subject와 payload 형식만 맞추면 Python SDK를 쓰든, Go SDK를 쓰든, 직접 NATS client를 쓰든 EDG와 통합할 수 있다.

SDK는 이 계약을 감싸는 도구다. 반복 작업을 줄이고, 재연결이나 collect loop 같은 공통 로직을 제공하지만, 플랫폼 통합의 유일한 입구는 아니다.

## Python SDK에서 Go SDK로

처음에는 Python SDK가 자연스러웠다. Modbus, BACnet, EtherNet/IP 같은 프로토콜을 다룰 때 Python 생태계가 빠르게 실험하기 좋기 때문이다.

하지만 엣지 환경에서는 단일 바이너리 배포나 Go-native 프로토콜 라이브러리가 더 편한 경우도 있다. 그래서 Go SDK를 추가하면서 Python SDK와 같은 표면을 맞추는 데 집중했다.

Go SDK가 제공해야 하는 기본 기능은 단순했다.

- 일정 주기로 값을 수집하는 `Collect` loop
- `TagValue` 모델과 quality/unit 표현
- NATS publish
- 자산과 관계 메타데이터 CRUD
- 메타데이터 변경 이벤트 구독
- 연결 실패와 재시도 처리

핵심은 "Go SDK가 별도 제품처럼 느껴지지 않게 하는 것"이었다. 언어는 달라도 EDG 입장에서 어댑터가 말하는 wire format은 같아야 한다.

## Modbus TCP reference adapter

SDK만 있으면 여전히 추상적이다. 그래서 실제 현장에서 가장 흔하게 만나는 Modbus TCP 예제를 Python과 Go 양쪽에 넣었다.

예제 어댑터는 YAML mapping을 읽어 레지스터를 `TagValue`로 바꾼다.

```yaml
version: 1
host: 127.0.0.1
port: 502
unit_id: 1
poll_interval: 1.0
timeout: 1.0

registers:
  - name: temperature
    function: holding
    address: 0
    type: int16
    scale: 0.1
    unit: "C"
  - name: flow_rate
    function: input
    address: 100
    type: float32
    word_order: CDAB
    unit: "L/min"
```

여기서 `word_order`가 중요했다. Modbus 장비는 32-bit 값을 다룰 때 word swap이 흔하고, 제조사 매뉴얼마다 표기가 다르다. `ABCD`, `CDAB`, `BADC`, `DCBA`를 명시적으로 지원하면 현장에서 디코딩 문제를 빨리 좁힐 수 있다.

Python 예제는 `pymodbus`, Go 예제는 `goburrow/modbus`를 사용한다. 둘 다 같은 YAML schema를 읽고 같은 EDG payload를 만든다.

## 테스트가 잡아준 부분

Modbus 디코더는 보기보다 실수하기 쉽다. 특히 signed/unsigned, float32, scale, word order 조합이 섞이면 "값은 나오는데 틀린 값"이 된다.

그래서 decoder matrix test를 양쪽 구현에 넣었다. 같은 testdata를 두 언어에서 공유하면 Python과 Go 구현이 같은 의미를 유지하는지 확인할 수 있다.

또 어댑터 SDK 쪽에는 collect loop, backoff, relation 모델, backward compatibility 테스트를 붙였다. 어댑터는 현장에서 오래 돌아가야 하므로, 예외가 발생했을 때 조용히 죽거나 잘못된 quality를 내보내면 안 된다.

## 배운 점

SDK를 먼저 크게 만들기보다, wire contract를 작게 고정하고 SDK를 그 위의 편의 계층으로 두는 편이 산업용 게이트웨이에는 더 잘 맞았다.

현장 프로토콜은 다양하고, 언어 선택도 상황마다 달라진다. EDG가 해야 할 일은 특정 언어를 강제하는 것이 아니라, 어떤 언어에서든 같은 subject와 payload로 들어올 수 있는 좁고 안정적인 입구를 제공하는 것이다.

관련 저장소: [github.com/e7217/edg](https://github.com/e7217/edg)
