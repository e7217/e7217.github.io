---
title: "Doorae에 내장 LLM 게이트웨이를 붙인 이유"
date: 2026-05-19 02:20:00 +0900
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

Doorae의 초기 에이전트 구조에서는 각 agent subprocess가 직접 LLM provider API를 호출했다. Claude Code는 Anthropic으로, Codex는 OpenAI로, 각 엔진이 자기 방식대로 나가는 구조였다. 단일 개발 머신에서는 이 방식이 단순하다. 하지만 여러 머신과 여러 엔진을 다루기 시작하면 한계가 드러난다.

이 문제를 풀기 위해 Doorae 서버 안에 LiteLLM 기반 LLM 게이트웨이를 내장하는 방향을 설계하고 구현했다.

## 직접 호출 구조의 한계

직접 호출 구조에는 세 가지 문제가 있었다.

첫째, 인터넷이 없는 머신에서는 에이전트를 돌리기 어렵다. 내부망에 있는 머신에서 Claude Code 에이전트를 띄우려면 그 머신이 `api.anthropic.com`에 나갈 수 있어야 한다. 머신마다 프록시와 시크릿을 배포하는 식으로 해결할 수는 있지만, 운영 부담이 크다.

둘째, 사용량 추적이 비어 있다. 호출이 Doorae 서버를 지나지 않기 때문에 어떤 룸의 어떤 에이전트가 어떤 모델을 얼마나 썼는지 알 수 없다.

셋째, 엔진별 프로토콜이 다르다. Anthropic `/v1/messages`와 OpenAI `/v1/chat/completions`를 모두 다루려면 단순 reverse proxy만으로는 부족하다.

## 선택한 구조

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

## 왜 app.mount가 아니라 subprocess인가

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

## 시크릿을 파일에 쓰지 않기

중요한 원칙은 API key 원문을 `litellm.yaml`에 쓰지 않는 것이다. config 파일에는 환경변수 참조만 들어간다.

```yaml
model_list:
  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet
      api_key: os.environ/DOORAE_LITELLM_ANTHROPIC_API_KEY
```

실제 값은 Doorae DB에 암호화해 두고, subprocess spawn 시점에만 복호화해 env로 주입한다. 이렇게 하면 설정 파일, 백업, 로그에 키가 평문으로 남는 위험을 줄일 수 있다.

## 에이전트 쪽 연결

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

## 배운 점

LLM 게이트웨이는 단순 프록시 기능이 아니었다. 네트워크가 막힌 머신을 지원하고, 사용량을 추적하고, 여러 엔진의 프로토콜 차이를 흡수하고, 시크릿 노출면까지 줄여야 했다.

Doorae에서는 이 기능을 별도 거대한 인프라로 빼기보다, 서버가 관리하는 작은 subprocess와 reverse proxy로 시작했다. 초기 단계에서는 이 방식이 배포 단순성과 운영 통제 사이의 균형이 좋았다.

관련 저장소: [github.com/e7217/doorae](https://github.com/e7217/doorae)
