---
title: "Doorae 룸 공유 파일을 에이전트 메모리로 배포하기"
date: 2026-05-19 02:10:00 +0900
categories:
  - ai
  - project-log
tags:
  - doorae
  - multi-agent
  - shared-files
  - memory
  - prompt-context
  - fastapi
---

Doorae에서 여러 에이전트와 협업하다 보면 자연스럽게 파일을 공유하고 싶어진다. 예를 들어 스펙 문서 하나를 올려두고 Claude Code와 Codex에게 동시에 의견을 받고 싶다. 하지만 초기 구조는 텍스트 메시지 중심이라, 사용자가 매번 문서 내용을 채팅에 복사해 넣어야 했다.

이 문제를 풀기 위해 룸 단위 공유 파일을 만들고, 그 파일을 참여 에이전트의 메모리 디렉터리로 복사 배포하는 구조를 잡았다.

## 요구사항

목표는 단순한 파일 업로드가 아니었다.

- 룸에 파일을 업로드하고 목록/삭제할 수 있어야 한다.
- 룸에 참여한 모든 에이전트가 같은 파일을 볼 수 있어야 한다.
- 서버 DB가 파일 본문 때문에 커지면 안 된다.
- 에이전트별 private memory와 룸 공유 자료의 생명주기를 분리해야 한다.
- 머신이나 에이전트가 나중에 들어와도 backfill이 가능해야 한다.

1차 범위는 텍스트 계열 파일로 제한했다. 크기도 256KB 이하로 잡았다. 이미지나 바이너리까지 바로 넣으면 토큰 예산과 저장소 정책이 동시에 복잡해진다.

## 저장 구조

파일 원문은 SQLite DB에 넣지 않았다. DB에는 metadata와 `sha256`만 저장하고, 원문은 `~/.doorae/room_files/<room_id>/` 아래에 둔다.

```text
~/.doorae/
├── doorae.db
├── room_files/
│   └── <room_id>/
│       ├── .tmp/
│       └── <file_id>
└── agents/
    └── <agent_id>/
        └── memory/
            ├── notes.md
            └── shared/
                └── spec.md
```

업로드는 임시 경로에 먼저 쓰고, sha256을 계산한 뒤 `os.replace`로 원자적 rename을 한다. DB commit이 실패하면 방금 쓴 파일을 지운다. 서버가 중간에 죽어 `.tmp`가 남아도, 부팅 시 `cleanup_orphans`가 정리한다.

이런 구조를 택한 이유는 명확하다. Doorae의 기본 저장소는 SQLite이고, 파일 원문을 DB에 넣으면 DB 파일이 빠르게 커지고 writer lock 부담도 커진다. 반면 `~/.doorae/`는 이미 agent memory와 machine 설정이 모이는 운영 루트라, 파일 저장소도 같은 루트 아래 두는 편이 관리하기 쉽다.

## 에이전트로 fan-out

업로드가 끝나면 서버는 룸에 배치된 에이전트마다 machine frame을 보낸다.

```text
AgentMemorySharedFileWriteFrame
  agent_id
  storage_name
  content
  content_sha256
```

machine daemon은 이 frame을 받아 각 에이전트의 `memory/shared/<storage_name>`에 파일을 쓴다. 이미 같은 sha256의 파일이 있으면 skip한다. 이 덕분에 재전송이 멱등이 된다.

삭제는 반대 방향이다. 서버에서 파일 metadata와 원문을 지우고, 각 에이전트에게 delete frame을 보낸다.

## 프롬프트 주입

파일이 디스크에만 있으면 에이전트가 모른다. 그래서 agent memory compose 단계에 `<shared-context>` 블록을 추가했다.

```xml
<shared-context>
  <file name="spec.md" sha256="...">
    ...
  </file>
</shared-context>
```

기존 `notes.md`는 에이전트 개인 메모리다. 반면 `memory/shared/`는 룸이 소유하는 정적 공유 자료다. 두 생명주기를 같은 파일에 섞지 않은 것이 중요했다. 개인 메모리는 에이전트가 쓰고 서버와 sync할 수 있지만, 공유 파일은 서버가 쓰고 machine이 배포하는 one-way 자료다.

## 메시지에서 파일 참조하기

사용자는 메시지 입력창에서 파일을 첨부하거나 `$filename` 형태로 공유 파일을 참조할 수 있다. 클라이언트는 이를 `metadata.references[]`로 보내고, 서버는 현재 룸에 실제로 존재하는 파일인지 canonicalize한다.

에이전트에게는 전체 파일을 메시지마다 다시 붙이지 않는다. 대신 turn-local hint만 준다.

```text
<referenced-files>
- spec.md: memory/shared/spec.md
</referenced-files>
```

실제 내용은 이미 `memory/shared/`와 `<shared-context>` 경로로 들어간다. 이렇게 해야 같은 파일을 여러 번 참조해도 프롬프트 구조가 과하게 커지지 않는다.

## 배운 점

멀티 에이전트 협업에서 파일 공유는 UI 기능처럼 보이지만, 실제로는 storage, machine protocol, prompt composition, access control이 함께 움직이는 기능이었다.

이번 작업에서 가장 중요한 결정은 DB가 아니라 디스크에 원문을 두고, 중앙 파일을 symlink하지 않고 에이전트별로 복사했다는 점이다. 저장 공간은 더 쓰지만, 각 에이전트의 작업 공간이 격리되고 실패 범위가 작아진다. MVP 단계에서는 이 단순함이 더 가치 있었다.

관련 저장소: [github.com/e7217/doorae](https://github.com/e7217/doorae)
