---
title: "EDG 자산 메타데이터와 변경 이벤트 설계"
date: 2026-05-19 01:10:00 +0900
categories:
  - smart-factory
  - project-log
tags:
  - edg
  - asset-model
  - metadata
  - digital-twin
  - nats
  - event
---

EDG를 단순한 시계열 수집기로만 보면 `asset_id`와 태그 값만 있으면 충분해 보인다. 하지만 스마트팩토리나 디지털 트윈 쪽으로 확장하려면 설비가 어떤 계층에 속하는지, 어떤 설비와 연결되는지, 외부 시스템의 식별자와 어떻게 매칭되는지가 중요해진다.

그래서 EDG에서는 데이터 수집과 별도로 **자산 메타데이터 모델**을 키우는 작업이 필요했다.

## 시작점

초기 데이터 payload는 단순하다.

```json
{
  "asset_id": "sensor-001",
  "values": [
    {
      "name": "temperature",
      "number": 25.5,
      "unit": "C",
      "quality": "good"
    }
  ]
}
```

이 구조는 수집에는 좋다. 그러나 운영 화면이나 상위 시스템에서는 "sensor-001이 어느 라인의 어떤 장비에 붙어 있는가"가 더 중요할 수 있다. 또 AAS, ECLASS, OPC UA node id 같은 외부 식별자를 보관할 자리도 필요하다.

그래서 자산에는 다음 개념을 추가했다.

- `source`: `manual`, `auto`, `modbus`, `opcua`, `mes`처럼 메타데이터 출처를 나타내는 값
- `external_ids`: AAS, ECLASS, IRDI, OPC UA node id 같은 외부 식별자
- `attributes`: 아직 색인까지는 필요 없는 어댑터별 부가 정보
- relations: `partOf`, `connectedTo`, `locatedIn` 같은 자산 관계

이 모델은 "태그 값 모음"에서 "설비 그래프의 일부"로 넘어가기 위한 기반이다.

## 자동 등록과 수동 거버넌스

EDG Core는 처음 보는 `asset_id`의 데이터가 들어오면 자산을 자동 등록할 수 있다. 현장 PoC에서는 이 방식이 편하다. 설비나 센서를 먼저 수동 등록하지 않아도 데이터가 들어오는 순간 최소 메타데이터가 생긴다.

하지만 모든 환경에서 자동 등록이 맞지는 않다. 운영 조직이 자산 마스터를 엄격하게 관리하거나, 승인되지 않은 설비 ID가 시스템에 생기면 안 되는 경우도 있다.

그래서 자산 등록 모드를 설정으로 분리했다.

```yaml
asset_registration:
  mode: auto
```

`auto` 모드에서는 미등록 자산을 `source: auto`로 생성한다. `manual` 모드에서는 미등록 자산을 자동 생성하지 않고 로그만 남긴다. 이 작은 설정 하나가 PoC 친화성과 운영 거버넌스 사이의 균형점을 만든다.

## 메타데이터 변경 이벤트

자산 정보가 바뀌면 다른 어댑터나 사이드카도 그 사실을 알아야 한다. 예를 들어 OPC UA 어댑터가 자산 목록을 캐싱하고 있다면, 수동 등록이나 관계 변경을 감지해 로컬 뷰를 갱신해야 한다.

EDG는 메타데이터 변경을 NATS subject로 발행한다.

```text
platform.meta.asset.changed
platform.meta.relation.changed
```

payload는 변경 전후 스냅샷을 담는다.

```json
{
  "schema_version": 1,
  "event_type": "created",
  "entity_type": "asset",
  "entity_id": "sensor-001",
  "source": "auto",
  "before": null,
  "after": {
    "id": "sensor-001",
    "name": "sensor-001",
    "source": "auto"
  }
}
```

여기서 중요한 점은 이 이벤트가 **best-effort notification**이라는 것이다. 데이터 플레인의 검증 스트림처럼 JetStream에 영구 저장되는 이벤트가 아니다. 구독자가 꺼져 있으면 놓칠 수 있다.

그래서 구독자는 시작할 때 먼저 `platform.meta.asset.list`로 현재 상태를 가져오고, 이후 `platform.meta.*.changed`를 구독해 증분만 적용해야 한다. 이벤트만 믿지 않고 항상 reconciliation 경로를 두는 방식이다.

## 설계하면서 정리된 원칙

이번 작업에서 정리된 원칙은 세 가지다.

첫째, 자산 메타데이터는 데이터 수집 payload와 분리한다. 수집 경로는 빠르고 단순해야 하고, 메타데이터는 점진적으로 풍부해져야 한다.

둘째, 자동 등록은 편의 기능이지 유일한 운영 모델이 아니다. 현장 실험에서는 자동 등록이 속도를 내지만, 실제 운영에서는 수동 거버넌스가 필요할 수 있다.

셋째, 메타데이터 이벤트는 상태 저장소가 아니라 알림이다. 이벤트를 놓쳐도 복구할 수 있도록 list API와 함께 써야 한다.

## 배운 점

스마트팩토리 시스템에서 "설비 데이터"는 값 자체보다 맥락이 중요해지는 순간이 온다. 온도 25.5도라는 값은 그 센서가 어느 설비의 어느 공정에 속하는지 알 때 의미가 커진다.

EDG의 메타데이터 작업은 디지털 트윈을 바로 완성하는 작업은 아니지만, 나중에 그 방향으로 갈 수 있도록 데이터 모델의 방향을 잡은 작업이었다.

관련 저장소: [github.com/e7217/edg](https://github.com/e7217/edg)
