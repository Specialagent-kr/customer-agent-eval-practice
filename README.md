# AI PM 코치 — 고객 응답 품질 검증 시스템

> ⚠️ **본 프로젝트는 AI 응답 품질 평가 파이프라인 구현 실습 예시입니다.**  
> 실제 서비스 운영용이 아닌, 학습 및 참고 목적으로 제작되었습니다.

---

## 프로젝트 개요

**AI PM 코치**(coach.proagent.kr) 서비스의 고객 문의 응답을 생성·평가·품질통제하는 전체 파이프라인을 Claude Code 환경에서 구현한 실습 프로젝트입니다.

```
고객 문의 입력
    │
    ▼
Customer Agent (응답 초안 생성)
    │
    ▼
Eval Agent (판정 트랙 + 기록 트랙 병렬 실행)
    │
    ├── ✅ Pass → 자동 발송
    │
    └── ❌ Fail → HITL (Slack DM 리뷰 요청)
                    │
                    ├── 승인 → 발송
                    ├── 수정 → 재검토
                    └── 반려 → 종료
```

---

## 파일 구조

```
Customer Agent/
│
├── CLAUDE.md                        # 전체 시스템 동작 정의 (Agent 지침서)
├── README.md                        # 본 파일
├── eval_log.md                      # Eval 결과 누적 기록 (자동 생성)
│
├── 예제파일/                         # Claude Knowledge 파일 (핵심 참조 문서)
│   ├── reference_knowledge.md       # 제품 지식 베이스 (Customer Agent 참조)
│   ├── eval_criteria.md             # 5차원 평가 기준표 (Rubric)
│   ├── test_cases.md                # Golden-Set 3개 + Edge Case 6개
│   ├── test_samples.md              # 테스트 입력 데이터 10개
│   ├── draft_prompt.md              # v0.1 초안 프롬프트 + 강사용 보완 포인트
│   ├── eval_execution.md            # 수동 평가표 + Judge LLM 프롬프트
│   └── quality_control.md          # Guardrail + HITL 워크플로우 가이드
│
├── .claude/
│   └── commands/                    # Claude Code 커스텀 명령어
│       ├── customer.md              # /customer — 응답 생성 + 자동 eval
│       ├── eval.md                  # /eval — 수동 평가 실행
│       ├── hitl.md                  # /hitl — Slack 스레드 읽고 액션 처리
│       ├── log.md                   # /log — 전체 Eval 로그 요약
│       └── compare.md              # /compare — 최근 2개 Eval 비교
│
└── project_plan.md                  # 프로젝트 전체 계획 및 실습 시나리오
```

---

## 사용 방법

### Claude Code에서 사용하기 (권장)

1. 이 저장소를 클론합니다.
   ```bash
   git clone https://github.com/Specialagent-kr/customer-agent-eval-practice.git
   cd customer-agent-eval-practice
   ```

2. Claude Code에서 프로젝트 폴더를 엽니다.
   ```bash
   claude
   ```

3. 채팅창에서 아래 명령어를 사용합니다.

| 명령어 | 설명 |
|--------|------|
| `/customer [고객 문의]` | 응답 초안 생성 + 자동 평가까지 한 번에 실행 |
| `/eval` | 직전 Customer Agent 응답을 수동으로 평가 |
| `/hitl EVAL-XXX` | Slack 스레드 답장을 읽어 승인/수정/반려 처리 |
| `/log` | 현재 대화의 전체 Eval 로그 요약 출력 |
| `/compare` | 가장 최근 2개 Eval 결과를 나란히 비교 |

#### 실행 예시

```
/customer 안녕하세요, PRD 기능이 어떻게 되나요?
```
→ Customer Agent가 응답 초안을 생성하고, Eval Agent가 자동으로 품질 평가합니다.

```
/customer 돈 내고 쓰는 서비스가 이 모양이에요??? 당장 환불해주세요!!!
```
→ Fail 판정 후 담당자에게 Slack DM이 자동 발송됩니다.

```
/hitl EVAL-002
```
→ Slack 스레드의 담당자 답장(승인/수정/반려)을 읽어 자동 처리합니다.

---

### Claude Desktop에서 사용하기

1. Claude Desktop을 열고 새 Project를 생성합니다.
   - 이름: `AI-PM-Coach-CS-Eval`

2. **Project Instructions**에 `CLAUDE.md` 전체 내용을 붙여넣습니다.

3. **Knowledge**에 아래 4개 파일을 업로드합니다.
   - `예제파일/reference_knowledge.md`
   - `예제파일/eval_criteria.md`
   - `예제파일/test_cases.md`
   - `예제파일/test_samples.md`

4. 채팅창에서 `/customer`, `/eval`, `/log`, `/compare` 명령어를 사용합니다.

> ⚠️ Claude Desktop에서는 `/hitl` 명령어(Slack 스레드 읽기)가 동작하지 않습니다.  
> Slack MCP 연동은 Claude Code 환경에서만 지원됩니다.

---

## 핵심 개념

### 두 트랙 병렬 구조

| 트랙 | 역할 | 판단 기준 |
|------|------|----------|
| 판정 트랙 (Gate) | 발송 여부 결정 | PII, 환불/결제 키워드, 강한 불만, 경쟁사 언급 |
| 기록 트랙 (Record) | 품질 데이터 축적 | 정확성, 형식 준수, 톤&공감, 완결성, 안전성 |

> **핵심**: 품질 점수(A등급)와 판정(Fail)은 독립적입니다.  
> "응답이 잘 쓰여졌더라도, 비즈니스 리스크가 있으면 사람이 확인해야 한다."

### Fail 판정 조건

아래 중 하나라도 해당하면 Fail → Slack DM 자동 발송:

- 🔴 PII 검출 (주민번호/카드번호/계좌번호/여권번호)
- 🟡 PII 검출 (전화번호/이메일)
- 환불/결제/카드/청구/취소/구독해지 키워드
- 강한 불만 (부정 감정 + 감탄/물음표 3개 이상)
- 응답에 "확인 후 안내" 류 표현
- 경쟁사 제품명 언급

### HITL 워크플로우

Fail 판정 시 담당자에게 Slack DM이 발송되며, 담당자는 스레드에 답장으로 액션을 선택합니다:

| 답장 | 동작 |
|------|------|
| `승인` | 응답을 그대로 고객에게 발송 |
| `수정 [내용]` | 수정사항 반영 후 재리뷰 |
| `반려` | 프로세스 종료, 수동 대응 전환 |

---

## 유사 프로젝트 직접 만들기 가이드

이 구조를 참고하여 자신만의 CS Eval 시스템을 만들 수 있습니다.

### 1단계: 도메인 정의

`reference_knowledge.md`를 자신의 서비스에 맞게 작성합니다.
- 제품/서비스 개요
- 핵심 기능 목록
- 자주 묻는 질문 (FAQ)
- 고객 지원 채널

### 2단계: 평가 기준 설계

`eval_criteria.md`를 서비스 특성에 맞게 커스터마이징합니다.
- 평가 차원과 가중치를 조정합니다. (예: 법률 서비스라면 정확성 가중치↑)
- 각 차원의 채점 기준(Rubric)을 구체적으로 작성합니다.

### 3단계: 테스트 케이스 정의

`test_cases.md`에 Golden-Set과 Edge Case를 정의합니다.
- **Golden-Set**: 빈번하게 발생하는 정상 문의 + 모범 응답(Ground Truth)
- **Edge Case**: 개인정보 포함, 강한 불만, 범위 밖 질문 등 + 반드시 지켜야 할 조건(MUST)

### 4단계: Fail 조건 설계

비즈니스 리스크 기준으로 HITL을 트리거할 조건을 정의합니다.
- **보안 리스크**: 어떤 PII를 차단할지
- **비즈니스 리스크**: 어떤 키워드/상황에서 사람의 확인이 필요한지
- **브랜드 리스크**: 경쟁사 언급, 민감한 발언 등

### 5단계: CLAUDE.md 작성

위 4단계를 통합하여 CLAUDE.md에 전체 시스템 동작을 정의합니다.
- Customer Agent 역할과 응답 규칙
- Eval Agent 판정/기록 트랙 로직
- Slack 등 외부 연동 설정

### 6단계: 커스텀 명령어 등록

`.claude/commands/` 폴더에 자신의 워크플로우에 맞는 명령어 파일을 작성합니다.

---

## 기술 스택

- **AI**: Claude (Anthropic) — Customer Agent + Eval Agent + Judge LLM
- **인터페이스**: Claude Code (CLI) / Claude Desktop
- **알림**: Slack MCP (HITL DM 발송 및 스레드 읽기)
- **로그**: 로컬 마크다운 파일 (`eval_log.md`)

---

## 라이선스

본 프로젝트는 실습 및 학습 목적으로 자유롭게 참고하실 수 있습니다.

---

_Made with Claude Code · [ProAgent](https://blog.proagent.kr)_
