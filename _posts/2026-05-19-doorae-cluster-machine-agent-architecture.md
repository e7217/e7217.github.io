---
title: "Doorae의 Cluster, Machine, Agent 구조 정리"
date: 2026-05-19 02:00:00 +0900
categories:
  - ai
  - engineering
tags:
  - doorae
  - multi-agent
  - websocket
  - scheduler
  - architecture
---

Doorae는 여러 AI 에이전트가 같은 룸에서 대화하고 작업할 수 있게 만드는 멀티 에이전트 채팅 플랫폼이다. 겉으로 보면 채팅 UI지만, 내부 구조는 단순한 채팅 서버보다 조금 더 복잡하다. 에이전트는 같은 서버 프로세스 안에서 실행되지 않고, 별도 머신에서 subprocess로 뜬 뒤 서버와 WebSocket으로 통신한다.

이 구조를 잡을 때 가장 중요했던 구분은 **데이터 평면과 제어 평면을 분리하는 것**이었다.

## 물리적 구성

Doorae의 기본 구성 요소는 세 가지다.

```text
doorae-cluster
  채팅 서버, REST API, Web UI, 스케줄러

doorae-machine
  각 호스트에 상주하는 daemon
  서버의 명령을 받아 에이전트 subprocess를 spawn/kill

doorae-agent
  claude-code, codex, gemini-cli, openhands 같은 엔진 어댑터
  룸에 접속해 메시지를 읽고 답변
```

여기서 Machine은 단순 논리 단위가 아니라 실제로 서버와 다른 PC, VM, GPU 서버일 수 있다. 서버는 "어떤 머신이 어떤 엔진을 실행할 수 있는가"를 보고, 적절한 머신에 에이전트 생성을 명령한다.

## WebSocket 두 종류

서버에는 성격이 다른 WebSocket 경로가 있다.

```text
/ws/rooms/{id}
  유저와 에이전트가 메시지를 주고받는 데이터 평면

/ws/machines/{id}
  machine daemon이 spawn/kill 명령을 받는 제어 평면
```

Machine daemon은 `/ws/machines/{id}`에 연결해 자기 capability를 보고하고, 서버의 명령을 기다린다. 반면 에이전트 subprocess는 별도로 `/ws/rooms/{id}`에 연결한다. 즉 daemon의 제어 연결과 agent의 대화 연결은 분리되어 있다.

이 분리가 중요하다. daemon은 에이전트를 관리하지만, 대화 메시지를 대신 중계하지 않는다. 에이전트는 서버에 독립적으로 참여하는 room participant가 된다.

## 왜 이렇게 나눴나

에이전트는 로컬 파일, MCP 서버, CLI 도구, 모델 SDK 등 다양한 런타임 자원에 접근한다. 이 자원을 모두 중앙 서버 안으로 끌어오면 서버가 너무 많은 책임을 갖게 된다.

반대로 에이전트를 각 머신에서 실행하면 다음 장점이 생긴다.

- 로컬 개발환경이나 GPU 서버처럼 머신별 특성을 활용할 수 있다.
- 에이전트 엔진이 subprocess로 격리된다.
- 서버는 메시지, 인증, 스케줄링, 상태 관리에 집중한다.
- 머신이 끊겨도 서버는 그 사실을 presence로 감지하고 UI에 반영할 수 있다.

최근에는 이 구조 위에서 `machine_online` 값을 agent response에 포함시키는 작업도 했다. DB에는 에이전트 상태가 `running`으로 남아 있어도, 실제 machine WebSocket이 끊긴 상태라면 UI에서는 unreachable/offline으로 보여야 하기 때문이다.

## 스케줄러의 역할

사용자가 에이전트를 생성하면 서버는 그 에이전트를 어느 Machine에 둘지 결정한다. 이때 서버는 Machine이 보고한 capability와 현재 배치 상태를 보고 spawn 명령을 보낸다.

```text
user request
  -> cluster scheduler
  -> selected machine
  -> machine daemon spawn
  -> agent subprocess
  -> room websocket join
```

이 흐름 덕분에 사용자는 "에이전트 생성"이라는 선언적 요청만 하고, 실제 프로세스 생성과 연결은 시스템이 처리한다.

## 데이터와 제어를 섞지 않기

이 아키텍처에서 가장 피하고 싶었던 것은 machine daemon이 모든 것을 대신 처리하는 구조였다. daemon이 메시지 중계까지 맡으면 장애 지점이 애매해지고, agent lifecycle과 room participant lifecycle이 섞인다.

그래서 daemon은 제어 평면만 담당한다.

- 에이전트 프로세스 생성
- 에이전트 프로세스 종료
- 메모리 파일 materialize
- 공유 파일 write/delete
- 머신 capability 보고

실제 대화는 agent가 직접 room WebSocket에 붙어 처리한다.

## 배운 점

멀티 에이전트 시스템을 만들 때 "에이전트를 어디서 실행할 것인가"는 초기에 정해야 하는 큰 결정이다. 중앙 서버 안에서 모두 실행하면 단순해 보이지만, 로컬 도구와 머신별 자원을 쓰기 시작하는 순간 한계가 온다.

Doorae는 서버를 오케스트레이터로 두고, Machine을 실행 단위로 두고, Agent를 대화 참여자로 분리했다. 이 구분이 이후 공유 파일, LLM 게이트웨이, OpenHands 엔진 추가 같은 작업의 기반이 되었다.

관련 저장소: [github.com/e7217/doorae](https://github.com/e7217/doorae)
