# 1TEAM-Recon-Subagent

팀원2(이나윤) — Recon Subagent 담당 산출물.

## 이 저장소에 있는 것

| 산출물 | 위치 | 내용 |
|---|---|---|
| Subagent 정의 | `.claude/agents/recon-agent.md` | 역할(공격 표면 인벤토리, exploit 금지), tool 제한(`Read`/`Grep`/`Glob`/`WebFetch`/`Bash` — 파일·검색·웹 도구 위주), 실행 방법, 출력 계약 |
| 진단 절차 skill | `.claude/skills/recon-endpoint-mapping/SKILL.md` | 크롤러 우선 실행 → 소스 대조 → 파라미터 위치/의미 2단 분류(Pass A/B) → 인증 요구사항 기록 → Attack Surface 표 생성 |
| 표준 출력 스키마 | `docs/request-seed-schema.md` | `RequestSeed`/`RequestParameter`/`ReconResult` 필드 정의 — 팀 공용 하네스(`dast-harness/dast_harness/agent_kit/contract.py`)의 실제 코드 기준, 문서용으로 재해석하지 않음 |
| `/agents` 등록·테스트 로그 | `docs/subagent-registration-test.md` | 등록 시도 결과(환경 한계 포함), 실제 타겟 대상 기능 테스트, 테스트 중 발견한 분류 버그와 수정 내역 |
| 증적 기록(evidence.csv) | `evidence/`, `scripts/` | recon이 남기는 시도 로그 — 아래 참고 |

## 증적을 남기는 과정 (evidence.csv)

recon은 exploit을 안 하지만, "이 엔드포인트가 인증 없이 열려 있는가" 같은
관측도 5명 공통 스키마로 기록한다. `evidence/evidence.csv`(스키마:
`timestamp,target,endpoint,agent,operator,caller,hypothesis,payload,observation,new_info,status,evidence_ref`)에
`agent=Recon`으로 한 행씩 append한다 — 손 편집 대신 `scripts/append_evidence.py`를
쓴다. 절차 전문은 `.claude/skills/evidence-logging/SKILL.md`(모든 agent가
공유하는 skill)에 있다.

**실제로 검증됨** — `evidence/evidence.csv`의 6행은 recon-agent를 vulnapp의
7개 라우트에 실제로 돌려서(비인증 probe마다 즉시 기록) 나온 결과다. recon
findings는 보통 `unconfirmed`로 남는다 — 실제 우회 확인은 injection-agent/
access-control-agent 몫이라는 걸 그대로 보여준다.

**이 과정에서 실제 버그 하나를 발견하고 고쳤다.** Windows Git Bash(MSYS2)가
`--endpoint "/admin"`처럼 순수 경로 모양 인자를 조용히 `C:/Program Files/Git/admin`으로
바꿔치기해서, 처음 6행이 전부 잘못된 endpoint 값으로 기록됐다. 원인을 찾아
`scripts/append_evidence.py`에 `MSYS_NO_PATHCONV=1` 사용을 명시하고 모든 예시
커맨드에 반영한 뒤 재기록해서 지금 CSV는 정상이다 — `evidence/README.md`에
전체 경위가 있다.

## 핵심 설계 결정

- **크롤러를 새로 만들지 않는다.** `dast-harness/dast_harness/agent_kit/recon.py`가
  이미 계약을 만족하는 정찰 에이전트다. recon-agent는 이걸 실행하고, 소스 코드
  대조와 Attack Surface 표 정리로 값을 더한다.
- **분류(location)와 스키마를 분리한다.** `RequestSeed`/`RequestParameter`에는
  "이건 injection 후보다" 같은 필드가 없다 — 그건 Attack Surface 표의 라우팅
  힌트일 뿐, 재사용 가능한 인벤토리 자체에 섞지 않는다.
- **파라미터 위치(location)만으로 후보 클래스를 정하지 않는다.** 실제 타겟에
  돌려본 결과 `?id=`(query)도 `/orders/{id}`(path)만큼 명백한 access-control
  후보임이 드러나서, "위치는 기법을 정하고 의미(객체 참조 여부)는 클래스를
  정한다"는 2단 분류(Pass A/B)로 skill을 고쳤다. `docs/subagent-registration-test.md`에
  전체 경위가 있다.

## 다른 저장소와의 관계

통합 저장소 [`SECURITY-1TEAM-Orchestrator-chain`](https://github.com/chhhd/SECURITY-1TEAM-Orchestrator-chain)에
같은 파일이 `injection-agent`/`access-control-agent`와 나란히 들어 있다. 이
저장소는 그중 recon 부분만 떼어낸 슬라이스다 — 경로 구조는 통합본과 동일하게
맞춰 놔서 그대로 병합 가능하다.

## 알아둘 것

- 이 세션(SDK/VS Code 확장 환경)에서는 `.claude/agents/*.md`가 Agent tool의
  invokable subagent 목록에 자동 등록되지 않는다 (실제로 시도해서 확인함,
  `docs/subagent-registration-test.md` 참고). 표준 터미널 Claude Code 세션에서는
  `/agents`로 정상 등록되어야 한다 — 이 저장소의 파일 자체는 유효한 정의다.
