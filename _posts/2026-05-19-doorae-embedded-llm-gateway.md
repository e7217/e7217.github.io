---
title: "여러 머신의 AI 에이전트를 위한 LLM 호출 경로 설계"
date: 2026-05-19 02:20:00 +0900
excerpt: "여러 머신에서 실행되는 AI 에이전트가 provider API를 직접 호출할 때 생기는 네트워크, 사용량 추적, 시크릿 문제를 LLM 게이트웨이로 정리한 기록."
categories:
  - ai
  - engineering
tags:
  - doorae
  - llm-gateway
  - litellm
  - proxy
  - secrets
  - multi-agent
---

AI 에이전트를 한 대의 개발 머신에서만 실행할 때는 provider API를 직접 호출해도 큰 문제가 없다. 하지만 여러 머신에서 여러 엔진을 동시에 실행하기 시작하면, 네트워크 경로와 시크릿 배포와 사용량 추적이 모두 흩어진다.

이 글은 개인 프로젝트인 **Doorae**에서 여러 머신의 AI 에이전트가 LLM을 호출하는 경로를 어떻게 정리했는지에 대한 기록이다. 결론은 Doorae 서버가 LiteLLM 기반 게이트웨이를 관리하고, 에이전트 호출을 `/api/v1/llm/*`로 모으는 구조였다.

## 요약

- 문제: agent subprocess가 각자 provider API를 직접 호출해 네트워크, 사용량, 시크릿 관리가 흩어졌다.
- 결정: Doorae server가 LiteLLM proxy를 subprocess로 띄우고 reverse proxy로 감싼다.
- 구현: gateway supervisor, draft-Apply 설정, agent manifest env 주입, token sentinel 치환을 연결했다.
- 남은 과제: 실제 네트워크가 분리된 B-machine 환경에서 end-to-end 검증이 더 필요하다.

## 프로젝트 맥락

Doorae의 초기 에이전트 구조에서는 각 agent subprocess가 직접 LLM provider API를 호출했다. Claude Code는 Anthropic으로, Codex는 OpenAI로, 각 엔진이 자기 방식대로 나가는 구조였다.

단일 개발 머신에서는 이 방식이 단순하다. 하지만 다음 같은 구성이 생기면 이야기가 달라진다.

```text
server machine
  - doorae-cluster
  - internet access

worker machine A
  - claude-code agent
  - internet access

worker machine B
  - codex or openhands agent
  - no outbound internet
```

worker machine B가 인터넷에 나갈 수 없다면 에이전트는 provider API를 직접 호출할 수 없다. 머신마다 별도 프록시를 세팅하고 API key를 배포할 수도 있지만, 운영 부담이 빠르게 커진다.

## 문제

직접 호출 구조에는 세 가지 문제가 있었다.

첫째, 인터넷이 없는 머신에서는 에이전트를 돌리기 어렵다. 내부망에 있는 머신에서 Claude Code 에이전트를 띄우려면 그 머신이 `api.anthropic.com`에 나갈 수 있어야 한다.

둘째, 사용량 추적이 비어 있다. 호출이 Doorae 서버를 지나지 않기 때문에 어떤 룸의 어떤 에이전트가 어떤 모델을 얼마나 썼는지 알 수 없다.

셋째, 엔진별 프로토콜이 다르다. Anthropic `/v1/messages`와 OpenAI `/v1/chat/completions`를 모두 다루려면 단순 reverse proxy만으로는 부족하다.

## 선택지

| 선택지 | 장점 | 단점 | 판단 |
|---|---|---|---|
| 에이전트 직접 호출 유지 | 가장 단순하다 | 네트워크 없는 머신, 사용량 추적, 시크릿 분산 문제가 남는다 | 제외 |
| LiteLLM을 별도 sidecar로 운영 | 구현 공수가 낮다 | 배포 아티팩트와 운영 문서가 늘어난다 | 제외 |
| LiteLLM FastAPI app을 mount | 프로세스가 하나로 보인다 | LiteLLM 공식 경로가 아니고 lifespan/config 경계가 불안하다 | 제외 |
| Doorae server가 LiteLLM subprocess를 관리 | 단일 서버 운영 철학을 크게 해치지 않고 gateway를 얻는다 | supervision 코드가 필요하다 | 선택 |

## 결정

선택한 구조는 Doorae server가 LiteLLM proxy를 subprocess로 관리하는 방식이다.

```text
agent
  -> /api/v1/llm/*
  -> doorae-server reverse proxy
  -> litellm subprocess on 127.0.0.1:4001
  -> upstream provider
```

LiteLLM은 외부에 노출하지 않고 `127.0.0.1:4001`에만 바인딩한다. 외부에서 접근 가능한 유일한 경로는 Doorae의 `/api/v1/llm/*` reverse proxy다.

이 구조의 장점은 Doorae의 단일 프로세스 운영 철학을 크게 깨지 않는다는 점이다. 별도 sidecar나 Postgres 기반 LiteLLM full feature를 도입하지 않고, Doorae server lifespan이 subprocess를 supervise한다.

## 구현 포인트

### 1. subprocess supervisor

처음에는 LiteLLM FastAPI app을 Doorae server에 mount하는 방법도 생각할 수 있다. 하지만 이 방식은 LiteLLM의 공식 배포 경로가 아니고, lifespan, middleware, config loading 경계가 버전 업마다 깨질 수 있다.

반대로 CLI subprocess는 LiteLLM이 공식적으로 제공하는 실행 경로다. Doorae는 프로세스를 띄우고, health check하고, 죽으면 backoff로 재시작하는 책임만 가진다.

상태 머신은 단순하게 잡았다.

```text
INIT -> STARTING -> RUNNING
RUNNING -> CRASHED -> STARTING
RUNNING -> RESTARTING -> STOPPED -> STARTING
STARTING -> FAILED
```

설정 변경은 즉시 반영하지 않고 draft-Apply 패턴으로 처리한다. admin이 모델이나 시크릿을 수정해도 바로 respawn하지 않고, Apply를 누를 때 새 config를 렌더링하고 subprocess를 재시작한다.

### 2. 시크릿을 파일에 쓰지 않기

중요한 원칙은 API key 원문을 `litellm.yaml`에 쓰지 않는 것이다. config 파일에는 환경변수 참조만 들어간다.

```yaml
model_list:
  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet
      api_key: os.environ/DOORAE_LITELLM_ANTHROPIC_API_KEY
```

실제 값은 Doorae DB에 암호화해 두고, subprocess spawn 시점에만 복호화해 env로 주입한다. 이렇게 하면 설정 파일, 백업, 로그에 키가 평문으로 남는 위험을 줄일 수 있다.

### 3. 에이전트 쪽 연결

게이트웨이만 있어서는 충분하지 않았다. 에이전트가 실제로 이 경로를 쓰도록 spawn manifest에도 값이 들어가야 했다.

예를 들어 `claude-code`에는 다음 값이 들어간다.

```text
ANTHROPIC_BASE_URL=<server>/api/v1/llm
ANTHROPIC_AUTH_TOKEN=@DOORAE_AGENT_TOKEN
```

`codex`에는 OpenAI 호환 경로를 넣는다.

```text
OPENAI_BASE_URL=<server>/api/v1/llm/v1
OPENAI_API_KEY=@DOORAE_AGENT_TOKEN
```

여기서 `@DOORAE_AGENT_TOKEN`은 sentinel이다. 서버는 agent token의 평문을 저장하지 않고 hash만 보관한다. 평문 token은 machine daemon이 spawn 시점에 알고 있으므로, machine 쪽에서 sentinel을 실제 token으로 치환한다.

agent process 안에서는 이 값들을 장시간 `os.environ`에 남기지 않는다. SDK 호출이 일어나는 짧은 구간에만 `secrets_in_env`로 노출하고, 호출이 끝나면 원래 환경으로 복원한다. Bash나 파일 읽기 도구가 `/proc/self/environ`을 통해 token을 볼 수 있는 시간을 줄이기 위한 조치다.

## 검증

이 작업은 여러 패키지 경계가 맞물려 있어 단위별로 나누어 검증했다.

- cluster: gateway feature flag가 켜졌을 때 spawn manifest에 엔진별 base URL과 token sentinel이 들어가는지 확인
- machine: `@DOORAE_AGENT_TOKEN` sentinel을 spawn 시점의 실제 agent token으로 정확히 치환하는지 확인
- agent: Claude Code와 Codex SDK 호출 구간에서만 env가 노출되고, 호출 후 기존 환경이 복원되는지 확인

이 검증의 핵심은 "게이트웨이가 있다"가 아니라, 에이전트의 실제 SDK 호출이 Doorae 인증 토큰을 들고 `/api/v1/llm/*`로 들어가도록 전체 경로가 닫혔는지 확인하는 것이었다.

## 남은 과제

아직 남은 작업도 있다.

- 실제 인터넷이 없는 worker machine에서 end-to-end 호출 검증
- 서버 전체 feature flag가 아니라 agent별 gateway 사용 여부 설정
- Usage 화면에서 모델별 가격표를 붙여 실제 비용 추정
- admin UI에서 secret test 버튼을 gateway 경로까지 연결

## 배운 점

LLM 게이트웨이는 단순 프록시 기능이 아니었다. 네트워크가 막힌 머신을 지원하고, 사용량을 추적하고, 여러 엔진의 프로토콜 차이를 흡수하고, 시크릿 노출면까지 줄여야 했다.

Doorae에서는 이 기능을 별도 거대한 인프라로 빼기보다, 서버가 관리하는 작은 subprocess와 reverse proxy로 시작했다. 초기 단계에서는 이 방식이 배포 단순성과 운영 통제 사이의 균형이 좋았다.

관련 자료:

- 저장소: [github.com/e7217/doorae](https://github.com/e7217/doorae)
- ADR: [docs/decisions/004-embedded-litellm-gateway.md](https://github.com/e7217/doorae/blob/main/docs/decisions/004-embedded-litellm-gateway.md)
- 설계 문서: [docs/design/12-llm-gateway.md](https://github.com/e7217/doorae/blob/main/docs/design/12-llm-gateway.md)
