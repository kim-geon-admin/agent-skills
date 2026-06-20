---
name: hermes-profile-check
description: Use when the user asks to inspect Hermes agent profiles, profile health, authentication, gateway status, Google/GWS/Gemini OAuth, OpenAI Codex connection, Codex runtime, or Korean requests such as 프로필 점검, 프로필 인증 체크, 프로필명 점검, 프로필명 체크, 인증 확인, 연결 확인.
---

# Hermes Profile Check

## Overview

Hermes agent 프로필의 인증, Gateway 실행 상태, Codex 연결 상태를 점검한다. CLI로 확인 가능한 항목은 먼저 명령으로 확인하고, 명령이 없거나 실패한 항목만 파일 근거로 보수적으로 판단한다.

이 스킬은 `/google-work-briefing`이나 다른 스킬 산출물을 근거로 삼지 않는다. Hermes CLI, Hermes profile home, 그리고 명시적인 사용자 입력만 근거로 사용한다.

## Quick Workflow

1. 요청 문장에서 대상 프로필을 식별한다. 사용자가 `프로필 점검`, `전체 점검`, `각각 점검`처럼 말하면 모든 프로필을 대상으로 한다. `coder 점검`, `work 체크`처럼 이름이 보이면 해당 프로필만 대상으로 한다.
2. 가능한 경우 `hermes profile list`로 프로필 목록을 확인한다. Gateway 전체 상태가 필요하면 `hermes gateway list`도 실행한다.
3. 프로필별 명령은 항상 `hermes -p <profile>`로 스코프를 고정한다. 기본 프로필도 가능하면 `default`로 명시한다.
4. CLI 결과가 없거나 모호하면 `HERMES_HOME`, `profiles/<name>`, `gateway.pid`, `gateway_state.json`, `auth/google_oauth.json`, `auth.json`, `config.yaml`을 보조 근거로 확인한다.
5. 결과는 한국어 요약 표와 프로필별 근거로 보고한다. 확인하지 못한 항목을 정상으로 추정하지 말고 `확인 필요` 또는 `미점검`으로 남긴다.

## CLI Checks

| 목적 | 우선 명령 | 판단 |
| --- | --- | --- |
| 프로필 발견 | `hermes profile list` | 목록에 있는 프로필과 사용자가 이름으로 지정한 프로필을 점검 범위로 삼는다. |
| Gateway 전체 상태 | `hermes gateway list` | 여러 프로필의 Gateway 실행 여부를 빠르게 비교한다. |
| 프로필 Gateway | `hermes -p <profile> gateway status --deep` | 실행 중이면 `정상`, 중지이면 `실패`, PID나 출력이 모호하면 `확인 필요`. |
| GWS/Google 인증 | `hermes -p <profile> auth status google-gemini-cli` | Hermes repo 기준 GWS 요청은 우선 Google/Gemini OAuth, Google Code Assist 인증으로 해석한다. |
| Codex 인증 | `hermes -p <profile> auth status openai-codex` | 로그인 상태와 만료, 재인증 필요 여부를 확인한다. |
| 프로필 상태 | `hermes -p <profile> status --deep` | provider, model, runtime, 환경 문제를 보조 근거로 사용한다. |
| Codex CLI | `codex --version` | `codex_app_server` runtime 또는 Codex CLI 연결을 의심할 때 설치 여부만 확인한다. |

명령 실패를 처리할 때는 exit code, 핵심 stderr/stdout 한 줄, 실패한 명령을 근거에 남긴다. 네트워크 오류, sandbox 오류, 권한 오류는 인증 실패로 단정하지 않는다.

## File Fallback

Hermes CLI를 실행할 수 없거나 특정 항목의 출력이 부족할 때만 파일을 본다.

| 항목 | 파일 근거 | 판단 규칙 |
| --- | --- | --- |
| 프로필 위치 | `HERMES_HOME`, 기본 Hermes root, `profiles/<name>` | 폴더와 `config.yaml`이 있으면 존재 가능성이 높다. 없으면 `확인 필요`. |
| Gateway | `gateway.pid`, `gateway_state.json` | PID가 있고 현재 프로세스가 살아 있으면 `정상`. PID만 있고 프로세스 확인 불가면 `확인 필요`. |
| GWS/Google 인증 | `auth/google_oauth.json` | `access`, `refresh`, `expires`, `email` 존재 여부만 본다. 만료됐지만 refresh token이 있으면 `주의`. |
| Codex 인증 | `auth.json` | `providers.openai-codex` 또는 credential pool의 `openai-codex` 존재 여부를 본다. |
| Codex 선택 상태 | `config.yaml` | `model.provider: openai-codex`이면 Codex provider 선택. `model.openai_runtime: codex_app_server`이면 Codex app-server runtime 의존. |

토큰, refresh token, access token, API key, client secret 같은 민감값은 절대 출력하지 않는다. 파일 존재, provider 이름, 만료 시각, 계정 email 정도만 필요한 경우에 제한적으로 요약한다.

## Judgment Rules

상태 라벨은 다음 중 하나로 통일한다.

| 라벨 | 의미 |
| --- | --- |
| `정상` | 명령 또는 강한 파일 근거로 현재 사용 가능하다고 판단됨. |
| `주의` | 인증 흔적은 있으나 만료, provider 미선택, refresh 필요, runtime 의존성 누락 가능성이 있음. |
| `실패` | 명령이 명확히 logged out, stopped, missing, invalid 상태를 반환함. |
| `확인 필요` | 근거가 모호하거나 CLI 실행, 프로세스 확인, 파일 접근이 제한됨. |
| `미점검` | 사용자 범위나 환경상 의도적으로 확인하지 않음. |

`인증 있음`, `현재 provider로 선택됨`, `실제 요청 가능함`을 같은 의미로 쓰지 않는다. 예를 들어 Codex 인증은 있으나 `config.yaml`에서 provider가 다르면 `주의`로 보고하고, provider가 `openai-codex`인데 인증이 없으면 `실패`로 보고한다.

## Output Format

기본 응답은 아래 구조를 따른다.

```markdown
# Hermes 프로필 점검
기준 시각: YYYY-MM-DD HH:mm TZ
확인 범위: default, coder

| 프로필 | GWS/Google 인증 | Gateway | Codex 연결 | 근거 | 조치 |
| --- | --- | --- | --- | --- | --- |
| default | 정상 | 실패 | 주의 | auth status, gateway status, config.yaml | hermes -p default gateway start |

## 세부 근거
### default
- HERMES_HOME: ...
- 실행한 명령: ...
- 판단: ...
- 다음 조치: ...
```

조치는 가능한 한 Hermes CLI 명령으로 제시한다. 예: `hermes -p <profile> auth add google-gemini-cli`, `hermes -p <profile> auth add openai-codex`, `hermes -p <profile> gateway start`, `hermes -p <profile> setup`.

## Extension Rules

나중에 Slack, Discord, MCP, cron, toolset 같은 점검이 추가되면 기존 표의 프로필 행을 유지하고 새 항목을 독립된 열 또는 세부 근거 섹션에 추가한다. 새 점검도 먼저 CLI, 그다음 파일 근거, 마지막에 보수적 판단 순서를 따른다.
