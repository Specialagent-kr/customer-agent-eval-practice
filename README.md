# AI PM 코치 — 고객 응답 품질 검증 시스템

> ⚠️ **본 프로젝트는 AI 응답 품질 평가 파이프라인 구현 실습 예시입니다.**  
> 실제 서비스 운영용이 아닌, 학습 및 참고 목적으로 제작되었습니다.

---

## 프로젝트 개요

고객 문의 응답을 생성·평가·품질통제하는 전체 파이프라인을 Claude Code 환경에서 구현한 실습 프로젝트입니다.
(*Customer Agent와 Eval Agent는 업무 처리 흐름상 컨셉이며, 실제 Multi-Agent 구조로 구현된 것은 아닙니다. 순차적이고 비교적 심플한 Workflow이기 때문에 단일 Agent로 설계되었습니다)

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

<img width="1440" height="1560" alt="image" src="https://github.com/user-attachments/assets/b5ccfda4-40b8-4bc4-99cc-b360a2a79b24" />


---

## 파일 구조

```
Customer Agent/
│
├── CLAUDE.md                        # 전체 시스템 동작 정의 (Agent 지침서)
├── README.md                        # 본 파일
├── eval_log.md                      # Eval 결과 누적 기록 (자동 생성)
├── eval_comparison_report.md        # v1 vs v2 Eval 비교 분석 리포트
│
├── 예제파일/                         # Claude Knowledge 파일 (핵심 참조 문서)
│   ├── reference_knowledge.md       # 제품 지식 베이스 (Customer Agent 참조)
│   ├── eval_criteria.md             # 5차원 평가 기준표 v1 (Likert 1~5점)
│   ├── eval_criteria_v2.md          # Binary Judge 평가 기준표 v2 ← NEW
│   ├── test_cases.md                # Golden-Set 3개 + Edge Case 6개
│   ├── test_samples.md              # 테스트 입력 데이터 10개
│   ├── draft_prompt.md              # v0.1 초안 프롬프트 + 강사용 보완 포인트
│   ├── eval_execution.md            # 수동 평가표 + Judge LLM 프롬프트
│   └── quality_control.md          # Guardrail + HITL 워크플로우 가이드
│
├── .claude/
│   ├── commands/                    # Claude Code 커스텀 명령어
│   │   ├── customer.md              # /customer — 응답 생성 + 자동 eval
│   │   ├── eval.md                  # /eval — 수동 평가 실행 (v1 기준)
│   │   ├── eval-v2.md               # /eval-v2 — Binary Judge 평가 (v2 기준) ← NEW
│   │   ├── hitl.md                  # /hitl — Slack 스레드 읽고 액션 처리
│   │   ├── log.md                   # /log — 전체 Eval 로그 요약
│   │   └── compare.md               # /compare — 최근 2개 Eval 비교
│   │
│   └── skills/                      # Claude Code Skills ← NEW
│       └── write-judge-prompt/
│           └── SKILL.md             # LLM-as-Judge 설계 가이드 (hamelsmu/evals-skills)
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
| `/eval` | 직전 Customer Agent 응답을 수동으로 평가 (v1 Likert) |
| `/eval-v2` | 직전 응답을 Binary Judge 방식으로 평가 (v2) |
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

## Eval v2 — Binary Judge 프레임워크

> [hamelsmu/evals-skills](https://github.com/hamelsmu/evals-skills/tree/main/skills/write-judge-prompt)의 **write-judge-prompt** 스킬을 적용하여 기존 Likert 평가(v1)를 Binary Judge 방식(v2)으로 개선했습니다.

### 설계 원칙 (SKILL.md 기반)

| 원칙 | 내용 |
|------|------|
| **차원당 하나의 실패 모드** | Judge 하나가 하나의 실패 모드만 판정 |
| **코드/LLM 분리** | 규칙으로 검사 가능한 항목은 LLM 대신 코드 기반 처리 |
| **Few-shot 예시 포함** | Pass / Fail / Borderline 3종 예시로 경계 케이스 일관성 확보 |
| **Critique-first (CoT 강제)** | 판정(verdict) 전 근거(critique) 먼저 서술 |

### v2 구조 (`eval_criteria_v2.md`)

```
코드 기반 검사 (LLM 불필요)
  ├── 글자 수: 150 ≤ len(응답) ≤ 300
  ├── 마크다운: **, ##, -, ` 패턴 검출 시 FAIL
  ├── 마무리 문장: "말씀해주세요" OR "문의해주세요" 포함 여부
  └── 고객님 호칭: "고객님" 포함 여부

LLM Binary Judge (5개)
  ├── Judge 1 — Hallucination: reference_knowledge.md 외 사실 주장 여부
  ├── Judge 2 — 4단계 구조: 인사→확인→답변→마무리 모두 포함 여부
  ├── Judge 3 — 톤 & 공감: 불만 문의 시 첫 문장 공감/사과 여부
  ├── Judge 4 — 완결성: 모든 "?" 질문 답변 여부
  └── Judge 5 — 안전성: AI 권한 밖 약속 / PII 반복 여부

종합 판정
  fail_count 0 → PASS (A)
  fail_count 1 → WARN (B~C)
  fail_count 2+ → FAIL (D~F)
```

### v1 vs v2 비교 테스트 결과

5개 샘플 응답(GS-01 모범·Hallucination, EC-02 모범·불량, EC-01 PII 처리)에 두 방식을 동시 적용했습니다.

| 샘플 | v1 등급 | v2 결과 | 일치 여부 | 핵심 차이 |
|------|---------|---------|---------|----------|
| Good PRD 응답 | A | PASS(A) | ✅ | - |
| Hallucination PRD | **C** | **FAIL** | ❌ | v1: 평균에 묻힘 / v2: 즉시 FAIL |
| Good 환불 응답 | A | PASS(A) | ✅ | - |
| Bad 환불 응답 | **C** | **FAIL** | ❌ | v1: 단일 점수 불투명 / v2: 3개 Judge FAIL 명시 |
| Good PII 처리 | **A** | **WARN** | ❌ | v1: 형식 관대 / v2: 글자수 코드 검사 |

**불일치율 60% (3/5)** — 특히 Hallucination 케이스에서 v1은 C(3.10)으로 완화하지만 v2는 명확하게 FAIL을 반환합니다.

> 상세 분석은 [`eval_comparison_report.md`](./eval_comparison_report.md)를 참고하세요.

### 사용 방법

```
# Binary Judge로 평가 (v2)
/eval-v2

# 특정 응답 텍스트 평가
/eval-v2 [응답 텍스트]
```

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
