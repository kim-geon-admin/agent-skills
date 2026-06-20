# Agent Skills

개인용/공개 배포용 Codex 및 Hermes agent skill 모음입니다.

현재 포함된 스킬은 `hermes-profile-check`입니다.

## Hermes Profile Check

`hermes-profile-check`는 Hermes agent의 프로필별 상태를 점검하기 위한 스킬입니다. 사용자가 프로필 인증, Google Workspace CLI 인증, Gateway 실행 여부, Codex 연결 상태를 확인해 달라고 요청할 때 사용합니다.

스킬 위치:

```text
skills/hermes-profile-check
```

## 설치

공개 GitHub repo에서 Codex skill installer로 설치할 수 있습니다.

```bash
install-skill-from-github.py --repo kim-geon-admin/agent-skills --path skills/hermes-profile-check
```

설치 후 Codex가 새 스킬 목록을 다시 읽도록 Codex 또는 VS Code를 재시작하세요.

## 트리거 예시

다음과 같은 요청에서 스킬이 사용되도록 작성되어 있습니다.

- `프로필 점검`
- `프로필 인증 체크`
- `전체 프로필 체크`
- `<프로필명> 점검`
- `<프로필명> 체크`
- `Google Workspace CLI 인증 상태 확인해줘`
- `GWS 인증 상태 확인해줘`
- `Gateway running 여부 봐줘`
- `Codex 연결 상태 확인해줘`

## 점검 항목

| 항목 | 우선 확인 방식 | 보조 확인 방식 |
| --- | --- | --- |
| 프로필 목록 | `hermes profile list` | `HERMES_HOME`, `profiles/<name>` |
| Google Workspace CLI 인증 | `hermes -p <profile> auth status google-workspace-cli` | Google Workspace CLI 인증 저장소 또는 `auth/google_oauth.json` |
| Gateway 상태 | `hermes gateway list`, `hermes -p <profile> gateway status --deep` | `gateway.pid`, `gateway_state.json` |
| Codex 인증/연결 | `hermes -p <profile> auth status openai-codex`, `codex --version` | `auth.json`, `config.yaml` |

## 판단 기준

스킬은 점검 결과를 다음 상태로 분류합니다.

| 상태 | 의미 |
| --- | --- |
| `정상` | 명령 또는 강한 파일 근거로 현재 사용 가능하다고 판단됨 |
| `주의` | 인증 흔적은 있으나 만료, provider 미선택, refresh 필요, runtime 의존성 누락 가능성이 있음 |
| `실패` | 명령이 logged out, stopped, missing, invalid 같은 명확한 실패 상태를 반환함 |
| `확인 필요` | 근거가 모호하거나 CLI 실행, 프로세스 확인, 파일 접근이 제한됨 |
| `미점검` | 요청 범위나 환경상 의도적으로 확인하지 않음 |

## 보안 원칙

스킬은 토큰, refresh token, access token, API key, client secret 같은 민감값을 출력하지 않도록 지시합니다. 필요한 경우 파일 존재 여부, provider 이름, 만료 시각, 계정 email 정도만 제한적으로 요약합니다.

## 출력 형식

기본 응답은 한국어 요약 표와 프로필별 세부 근거를 포함합니다.

```markdown
# Hermes 프로필 점검
기준 시각: YYYY-MM-DD HH:mm TZ
확인 범위: default, coder

| 프로필 | Google Workspace CLI | Gateway | Codex 연결 | 근거 | 조치 |
| --- | --- | --- | --- | --- | --- |
| default | 정상 | 실패 | 주의 | auth status, gateway status, config.yaml | hermes -p default gateway start |
```

## 관련 파일

- [`skills/hermes-profile-check/SKILL.md`](skills/hermes-profile-check/SKILL.md)
- [`skills/hermes-profile-check/agents/openai.yaml`](skills/hermes-profile-check/agents/openai.yaml)
