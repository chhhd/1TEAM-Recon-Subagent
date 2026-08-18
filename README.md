# 1TEAM-Recon-Subagent

팀원2(이나윤) — Recon Subagent 담당 산출물.

> 이 저장소는 두 갈래 작업을 하나로 합친 결과다: 5인 팀 공용 산출물 체계(`evidence.csv`,
> `/agents` 등록 테스트 기록 등, 원래 이 저장소가 갖고 있던 틀)를 **큰 틀**로 유지하면서,
> recon-agent 자체의 설계·절차·검증 도구는 별도로 진행됐던 [`recon_subagent`](https://github.com/nyl23/recon_subagent)
> 프로젝트의 더 상세하고 실전 검증된 버전으로 **교체**했다. 아래 "병합 경위" 절 참고.

## 이 저장소에 있는 것

| 산출물 | 위치 | 내용 |
|---|---|---|
| Subagent 정의 | `.claude/agents/recon-agent.md` | 역할(공격 표면 인벤토리, exploit 금지) · Safety Gate · 입출력 계약 · evidence.csv 기록 지침 |
| 진단 절차 skill | `.claude/skills/recon/SKILL.md` | PTES Reconnaissance / OWASP WSTG Information Gathering 기반 Step 0~7 체크리스트 (정보수집 → 서비스/포트 식별 → 기술스택 추정 → Endpoint 탐색 → Attack Surface 정리 → 취약점 후보 생성) |
| 출력 스키마 | `.claude/skills/recon/reference/attack-surface-schema.md` | `Target / Observed / Potential Attack Surface` 3단 텍스트 포맷(후보 A') — 신뢰도 표기 필수, endpoint 표기 일관성 강제 |
| 저장 경로 규칙 | `.claude/skills/recon/reference/output-path-convention.md` | `recon-output/<target-slug>/attack-surface.<ext>`(최신본) + `runs/<timestamp>/`(매 실행 영구 보존) |
| 도구 단일 기준 정의 | `.claude/skills/recon/reference/tools.json` | curl/nslookup/whois/nmap/ffuf의 실행 위치·호출 형식·플래그·wordlist·제한값 (Single Source of Truth) |
| 권한 설정 | `.claude/settings.json` | `tools.json`에서 `tools/sync_permissions.py`로 자동 생성되는 Bash 허용 목록 + evidence 기록 스크립트 |
| 공용 skill: 증적 기록 | `.claude/skills/evidence-logging/SKILL.md` | 5명(Recon/Injection/IDOR/Auth/CVE) 전원이 공유하는 `evidence.csv` 스키마·기록 절차 |
| 증적 기록(evidence.csv) | `evidence/`, `scripts/` | recon이 남기는 시도 로그 — 아래 "증적을 남기는 과정" 참고 |
| 로컬 연습용 랩 타겟 | `lab-target/` | `127.0.0.1` 전용 더미 취약 웹앱 2종 (`app.py`, `app2.py`) — recon-agent 실전 검증용 |
| 자가 검증·동기화 도구 | `tools/` | `validate_attack_surface.py`(출력 포맷 자가 검증), `sync_permissions.py`(tools.json → settings.json 동기화), `wordlists/common.txt`(ffuf용 165개 wordlist) |
| 팀 공유용 요약 가이드 | `docs/recon-agent-guide.md` | "지금 이 agent가 뭘 하는지"만 5분 안에 정리한 문서 — 처음 공유받았다면 이것부터 |
| (레거시) 초기 설계 문서 | `docs/request-seed-schema.md`, `docs/subagent-registration-test.md` | 병합 전 1차 설계(`dast-harness` 재사용 + `RequestSeed` 스키마) 당시 문서. 현재 recon-agent는 이 설계를 쓰지 않지만, `/agents` 등록 한계 등 여전히 유효한 관측이 있어 보존 — 아래 "병합 경위" 참고 |

## 병합 경위

이 저장소는 원래 recon-agent를 `dast-harness/dast_harness/agent_kit/recon.py`(팀 공용 하네스)를
실행하고 그 결과(`RequestSeed`)를 소스 코드 대조로 보강하는 방식으로 설계했었다
(`docs/request-seed-schema.md`, `docs/subagent-registration-test.md`가 그 시점 기록).

이후 별도로 진행된 [`recon_subagent`](https://github.com/nyl23/recon_subagent) 프로젝트에서
같은 recon-agent를 **nmap/ffuf/curl 등 도구를 직접 호출하는 독립 구현**으로 다시 만들고,
실제 대상(더미 랩·DVWA) 6회 실행으로 검증하며 20여 건의 하네스 엔지니어링 결정을 남겼다
(상세 시간순 기록은 원본 저장소 README 참고). 이 버전이 더 상세하고 실전 검증도 많이
거쳤다고 판단해, **recon-agent의 설계·절차·스키마·도구는 `recon_subagent` 쪽을 그대로
채택**하고, 이 저장소가 갖고 있던 **5인 팀 공용 자산(evidence.csv 로깅 체계)은 유지한 채
recon-agent.md에 다시 연결**했다:

- `.claude/agents/recon-agent.md`, `.claude/skills/recon/`, `lab-target/`, `tools/`,
  `docs/recon-agent-guide.md`, `.claude/settings.json`, `.gitignore` — `recon_subagent`에서 그대로 이식
- `.claude/skills/evidence-logging/`, `evidence/`, `scripts/` — 이 저장소의 것을 유지
- `.claude/agents/recon-agent.md`에는 `recon_subagent` 원본 본문 그대로에 "## 5. 증적 기록
  (evidence.csv)" 절을 새로 추가해서, 인증 요구 여부를 확인하는 조회마다
  `scripts/append_evidence.py`로 `agent=Recon` 행을 남기도록 연결했다 (§7 완료 조건에도 반영)
- `.claude/skills/recon-endpoint-mapping/`(RequestSeed 기반 구버전 절차)은 `recon` skill로
  대체되어 삭제, `docs/request-seed-schema.md`/`subagent-registration-test.md`는 레거시로 보존

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

## recon-agent 설계 (recon_subagent 채택분)

- **판단 주체가 아니라 정보 생산자다.** 취약점을 확정하거나, 다음에 어떤 Agent를
  부를지 결정하지 않는다. 후속 Agent(Injection / IDOR·Authorization / Web Logic)가
  바로 쓸 수 있는 구조화된 Attack Surface만 만든다.
- **수동 탐색 우선, 능동 도구는 보강용.** `robots.txt`/링크/JS 문자열 등으로 먼저
  찾고, 부족할 때만 `ffuf`(wordlist·스레드·타임아웃은 `tools.json` 고정값)로 승격한다.
- **Safety Gate.** Claude Code 체크포인트가 되돌릴 수 없는 외부 부작용(네트워크
  요청)에 대응하기 위해, scope 확인 → 조회성 요청만 허용 → 모든 능동 요청을
  `execution_log.jsonl`에 기록 → rate 제한을 지킨다.
- **세션 독립성 대응.** 호출마다 새 컨텍스트로 시작하므로 결과를 반드시
  `recon-output/<target-slug>/`에 파일로 영속화하고, 대화 기록에 의존하지 않는다.
- **출력은 `Target/Observed/Potential Attack Surface` 3단 텍스트(후보 A').**
  신뢰도(`강함`/`보통`/`약함`) 표기 필수, `Observed`와 `Potential Attack Surface`의
  endpoint 문자열을 동일하게 써서 Orchestrator가 문자열 매칭만으로 다음 Agent에게
  최소 컨텍스트를 추릴 수 있게 한다. `tools/validate_attack_surface.py`로 저장 직후
  자가 검증한다.
- **도구 정의는 `tools.json` 한 곳에 모은다.** `recon-agent.md`/`SKILL.md`/`settings.json`이
  각자 도구 세부값을 반복해서 적지 않도록, `tools/sync_permissions.py`로
  `settings.json`을 자동 동기화한다.

## 다른 저장소와의 관계

통합 저장소 [`SECURITY-1TEAM-Orchestrator-chain`](https://github.com/chhhd/SECURITY-1TEAM-Orchestrator-chain)에
같은 파일이 `injection-agent`/`access-control-agent`와 나란히 들어 있다. 이
저장소는 그중 recon 부분만 떼어낸 슬라이스다.

recon-agent 자체의 설계 시행착오(20여 건의 결정 로그, 6회 실행 테스트 요약)는
원본 [`recon_subagent`](https://github.com/nyl23/recon_subagent) 저장소 README에
시간순으로 남아 있다 — 여기서는 중복하지 않는다.

## 알아둘 것

- 이 세션(SDK/VS Code 확장 환경)에서는 `.claude/agents/*.md`가 Agent tool의
  invokable subagent 목록에 자동 등록되지 않는다 (실제로 시도해서 확인함,
  `docs/subagent-registration-test.md` 참고). 표준 터미널 Claude Code 세션에서는
  `/agents`로 정상 등록되어야 한다 — 이 저장소의 파일 자체는 유효한 정의다.
