# AI Eval 실습 프로젝트 계획서

## 프로젝트 개요

**프로젝트명**: AI PM 코치 — 고객 문의 응답 품질 검증 시스템  
**목적**: Claude Desktop Application의 Project 기능을 활용하여, 고객 응답 생성 → 평가 → 품질 통제의 전체 파이프라인을 구현  
**대상**: AX 코칭 수강생 (기획/세일즈/마케팅 등 비개발 직군)

---

## 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                   Claude Desktop Project                     │
│                                                              │
│  📄 CLAUDE.md (프로젝트 지침서)                               │
│  📄 reference_knowledge.md (제품 지식 베이스)                  │
│  📄 eval_criteria.md (평가 기준표)                            │
│  📄 test_cases.md (Golden-Set & Edge Cases)                  │
│                                                              │
│   /customer + 문의                       /eval               │
│        │                                  │                  │
│        ▼                                  │                  │
│  ┌──────────────┐                         │                  │
│  │Customer Agent│─── 응답 초안 ────────────┤                  │
│  └──────────────┘                         │                  │
│                                           ▼                  │
│                    ┌─────────── Eval Agent Pipeline ────────┐│
│                    │                                        ││
│                    │  판정 트랙 (Gate)   기록 트랙 (Record)   ││
│                    │  ┌────────────┐   ┌────────────────┐   ││
│                    │  │PII Guardrail│   │Judge LLM 평가   │   ││
│                    │  │     ↓      │   │      ↓         │   ││
│                    │  │ 판정       │   │Eval Log 기록    │   ││
│                    │  │Pass │ Fail │   │      ↓         │   ││
│                    │  │  ↓  │  ↓   │   │평가 리포트      │   ││
│                    │  │발송 │ HITL  │   └────────────────┘   ││
│                    │  │     │ Slack │                        ││
│                    │  │  ↓  │리뷰  │                        ││
│                    │  │고객 │승인   │                        ││
│                    │  │응답 │수정   │                        ││
│                    │  │전달 │반려   │                        ││
│                    │  └────────────┘                        ││
│                    └────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

---

## 에이전트 구성

### Agent 1: Customer Agent (고객 답변 제작)

| 항목 | 내용 |
|------|------|
| 역할 | 고객 문의를 받아 참조 문서 기반으로 응답 초안 생성 |
| 입력 | `/customer` + 고객 문의 텍스트 |
| 참조 | reference_knowledge.md |
| 출력 | 응답 초안 텍스트 |
| 프롬프트 버전 | v0.1(실습용 60점) → v0.2(개선 후) |

### Agent 2: Eval Agent (고객 답변 평가)

| 항목 | 내용 |
|------|------|
| 역할 | Customer Agent의 응답을 평가하고 품질 통제 워크플로우 수행 |
| 입력 | `/eval` + (직전 응답 또는 평가할 텍스트) |
| 참조 | eval_criteria.md, test_cases.md |
| 출력 | 판정 결과 + 평가 리포트 + Eval Log |

Eval Agent는 두 개의 트랙을 병렬 실행합니다:

| 트랙 | 역할 | 구성 |
|------|------|------|
| 판정 트랙 (Gate) | 발송 여부 결정 | PII Guardrail → 판정(Pass/Fail) → Fail 시 HITL |
| 기록 트랙 (Record) | 품질 데이터 축적 | Judge LLM 평가 → Eval Log 기록 (항상 실행) |

---

## Claude Desktop 프로젝트 세팅 가이드

### Step 1. 프로젝트 생성
1. Claude Desktop 열기
2. 좌측 사이드바 > "Projects" > "Create Project"
3. 프로젝트 이름: `AI-PM-Coach-CS-Eval`

### Step 2. 프로젝트 지침 등록
1. Project Instructions에 `CLAUDE.md` 내용을 붙여넣기

### Step 3. Knowledge 파일 업로드

| 파일 | 용도 |
|------|------|
| `reference_knowledge.md` | Customer Agent가 참조하는 제품 지식 |
| `eval_criteria.md` | Eval Agent가 사용하는 평가 기준표 |
| `test_cases.md` | Golden-Set & Edge Case 정의 |
| `test_samples.md` | 사전 점검용 테스트 입력 데이터 |

### Step 4. Slack 연동 (MCP)
1. Claude Desktop 설정 > Integrations > Slack 연결
2. 리뷰 채널 `#cs-ai-review` 생성

---

## 실습 시나리오 (워크숍 진행 순서)

### 시나리오 A: Pass 흐름 — 자동 발송

```
[수강생] /customer 안녕하세요, PRD 작성 기능이 어떻게 되나요?

[Customer Agent] → 응답 초안 생성

[수강생] /eval

[Eval Agent]
  판정 트랙: PII 없음 → Pass ✅ → 자동 발송
  기록 트랙: 4.2/5.0 (B등급) → 로그 기록
  → 고객 응답이 자동 발송됩니다.
```

### 시나리오 B: Fail → HITL 승인 흐름

```
[수강생] /customer 돈 내고 쓰는 서비스가 이 모양이에요??? 환불해주세요!

[Customer Agent] → 응답 초안 생성

[수강생] /eval

[Eval Agent]
  판정 트랙: PII 없음, 환불+불만 키워드 → Fail ❌ → HITL 발동
  기록 트랙: 3.2/5.0 (C등급) → 로그 기록
  → Slack 리뷰를 보낼까요? (yes)

[담당자] Slack에서 응답 확인 → ✅ 승인
  → 고객 응답 전달
```

### 시나리오 C: Fail → HITL 수정 루프

```
[수강생] /customer 카드번호 1234-5678-9012-3456 결제 오류입니다.

[Customer Agent] → 응답 초안 (PII 포함 가능)

[수강생] /eval

[Eval Agent]
  판정 트랙: PII 검출 🔴 → Fail ❌ → HITL 발동
  기록 트랙: 2.1/5.0 (D등급) → 로그 기록
  → Slack 리뷰를 보낼까요? (yes)

[담당자] Slack에서 확인 → ✏️ 수정 (PII 제거 후 재작성)
  → 수정본 재리뷰 → ✅ 승인
  → 고객 응답 전달
```

### 시나리오 D: 프롬프트 개선 효과 비교

```
1. 초안 프롬프트(v0.1)로 EC-06(복합 시나리오) 실행 → /eval
2. 개선 프롬프트(v0.2)로 동일 케이스 실행 → /eval
3. /compare로 두 결과 비교
   → v0.1: Fail (PII+환불) / 2.1점 D등급
   → v0.2: Fail (환불) / 4.0점 B등급 (PII는 해결됨!)
```

---

## 파일 목록 총정리

### 실습 예제 파일 (커리큘럼 순서)

| 순서 | 파일명 | 커리큘럼 단계 | 용도 |
|------|--------|-------------|------|
| 1 | `reference_knowledge.md` | 사전 준비 | 제품 지식 베이스 (프롬프트 Input) |
| 2 | `draft_prompt.md` | 사전 점검 | 60점 초안 프롬프트 + 강사용 보완 포인트 |
| 3 | `test_samples.md` | 사전 점검 | 테스트 입력 데이터 10개 + 관찰 포인트 |
| 4 | `eval_criteria.md` | Eval 준비 | 5차원 평가 기준표 (Rubric) |
| 5 | `test_cases.md` | Eval 준비 | Golden-Set 3개 + Edge Case 6개 |
| 6 | `eval_execution.md` | Eval 실행 | 수동 평가표 + Judge LLM 프롬프트 + 비교 가이드 |
| 7 | `quality_control.md` | 품질 통제 | Guardrail + HITL 워크플로우 + 실습 가이드 |

### Claude Desktop 구현 파일

| 파일명 | Claude Desktop 등록 위치 | 용도 |
|--------|------------------------|------|
| `CLAUDE.md` | Project Instructions | 프로젝트 지침서 (전체 시스템 동작 정의) |
| `reference_knowledge.md` | Knowledge | 제품 지식 베이스 |
| `eval_criteria.md` | Knowledge | 평가 기준표 |
| `test_cases.md` | Knowledge | Golden-Set & Edge Cases |
| `test_samples.md` | Knowledge | 테스트 입력 데이터 |

### 강사 참고 파일

| 파일명 | 용도 |
|--------|------|
| `customer_agent_prompt.md` | v0.1(초안) / v0.2(개선) 프롬프트 비교 + 디브리핑 가이드 |
| `eval_agent_prompt.md` | 판정 트랙 + 기록 트랙 상세 로직 + 예상 결과표 |
| `project_plan.md` | 프로젝트 전체 계획 + 세팅 가이드 + 실습 시나리오 |
