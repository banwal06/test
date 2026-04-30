# 주식 종목 분석 AI 에이전트 시스템

> Claude Code 서브에이전트 기반 오케스트레이션 아키텍처
> GPT 자동매매 앱 연동 지원

## 시스템 구성: 2가지 모드

### 모드 A — 심층 분석 (`stock-analyst`)
개별 종목을 증권사 리서치 수준으로 분석하고 투자의견을 제시합니다.

### 모드 B — 일일 매매 유니버스 결정 (`trading-advisor`) ← GPT 자동매매 연동용
내 종목 vs AI 추천 종목을 비교하여 그날의 매매 리스트를 JSON으로 출력합니다.

---

## 아키텍처 개요 — 모드 B (자동매매 연동)

```
┌──────────────────────────────────────────────────────────┐
│  사용자: "오늘 삼성전자, NVIDIA 보고 있어"                    │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│           🔵 trading-advisor (Orchestrator)               │
│           역할: 내 종목 vs AI 추천 비교 → 매매 유니버스 결정    │
└───────┬──────────────────────────┬───────────────────────┘
        │                          │
   Phase 1 (병렬)             Phase 1 (병렬)
        │                          │
        ▼                          ▼
┌───────────────┐    ┌────────────────────────────────┐
│ 🩵 daily-      │    │ 사용자 종목 분석 (종목별 병렬)     │
│ screener      │    │                                │
│               │    │  🟢 profiler  🟡 financial       │
│ AI 추천 TOP 5  │    │  🟠 momentum                    │
│ (100점 스코어링)│    │                                │
└───────┬───────┘    └──────────────┬─────────────────┘
        │                          │
        └────────────┬─────────────┘
                     ▼
           ┌───────────────────┐
           │ 🩷 stock-comparator│
           │                   │
           │ 동일 기준 스코어링   │
           │ Head-to-Head 비교  │
           │ 최종 유니버스 결정   │
           └─────────┬─────────┘
                     │
           ┌─────────┴─────────┐
           ▼                   ▼
  📊 사용자 브리핑         📄 JSON 출력
  (사람이 읽는 용도)       (자동매매 앱 전달용)
           │                   │
           │                   ▼
           │         ┌──────────────────┐
           │         │ 🤖 GPT 자동매매 앱 │
           │         │ (기존 시스템)       │
           │         │                  │
           │         │ JSON 파싱 →       │
           │         │ 종목 매매 실행      │
           │         └──────────────────┘
           │
           ▼
   사용자 최종 확인/오버라이드
```

## 에이전트 구성

### 오케스트레이터 (2개)

| 파일명 | 역할 | Model | Color |
|--------|------|-------|-------|
| `stock-analyst.md` | 심층 분석 오케스트레이터 | Opus | 🔵 Blue |
| `trading-advisor.md` | **일일 매매 유니버스 결정 (GPT 연동)** | Opus | 🔵 Blue |

### 서브에이전트 (7개)

| 파일명 | 역할 | Model | Color |
|--------|------|-------|-------|
| `company-profiler.md` | 기업 개요 분석 | Sonnet | 🟢 Green |
| `financial-analyzer.md` | 재무 정량 분석 | Sonnet | 🟡 Yellow |
| `industry-analyzer.md` | 산업/경쟁 분석 | Sonnet | 🟣 Purple |
| `momentum-analyzer.md` | 모멘텀/수급 분석 | Sonnet | 🟠 Orange |
| `risk-assessor.md` | 리스크 종합 평가 | Sonnet | 🔴 Red |
| `daily-screener.md` | **일일 종목 스크리닝** | Sonnet | 🩵 Cyan |
| `stock-comparator.md` | **내 종목 vs AI 비교 판정** | Sonnet | 🩷 Pink |

## 설치 방법

### 프로젝트 레벨 (팀 공유)

```bash
mkdir -p .claude/agents
cp agents/*.md .claude/agents/
```

### 유저 레벨 (개인 전역)

```bash
mkdir -p ~/.claude/agents
cp agents/*.md ~/.claude/agents/
```

## 사용 방법

### 모드 A — 심층 분석

```
삼성전자 심층 분석해줘
```
```
Use the stock-analyst agent to analyze NVIDIA
```

### 모드 B — 일일 매매 유니버스 결정 (GPT 연동)

```
오늘 내가 보는 종목: 삼성전자, NVIDIA, 카카오
코스피+나스닥 스윙 기준으로 AI 추천이랑 비교해줘
```
```bash
# CLI에서 직접 실행
claude --agent trading-advisor
```

### 간편 모드 — 빠른 판정

```
카카오 오늘 사도 될까?
```

## GPT 자동매매 앱 연동

자세한 연동 방법은 **`INTEGRATION_GUIDE.md`** 를 참조하세요.

3가지 연동 방식을 지원합니다:
1. **파일 기반** — JSON 파일 저장 → GPT 앱이 읽기 (가장 간단)
2. **Claude API 직접 호출** — GPT 앱에서 Claude API를 호출 (완전 자동화)
3. **GPT↔Claude 브리지** — 기존 GPT 앱 시스템 프롬프트에 Claude 결과 주입 (최소 수정)

### 핵심 흐름

```
08:00  trading-advisor 실행 (내 종목 + AI 스크리닝)
08:30  JSON 생성 → GPT 자동매매 앱에 전달
09:00  장 개시 → GPT 앱이 universe 기반으로 매매 실행
15:30  장 마감 → 결과 기록
```

## 분석 리포트 구조

최종 리포트는 다음 6개 섹션으로 구성됩니다:

1. **기업 개요** — 비즈니스 모델, 경쟁력, 경영진 평가
2. **재무 분석** — 수익성/성장성/안정성 + 밸류에이션
3. **산업 분석** — 시장 규모, 경쟁 구도, 트렌드
4. **모멘텀 분석** — 주가 트렌드, 수급, 촉매, 기술적 분석
5. **리스크 요인** — 리스크 매트릭스 + 시나리오 분석
6. **종합 의견** — 투자 의견 + 목표 주가 + 추천 근거 + 리스크 Top 3

## 커스터마이징

### MCP 서버 연동 (선택)

금융 데이터 API가 있다면 서브에이전트에 MCP 서버를 추가할 수 있습니다:

```yaml
# financial-analyzer.md frontmatter에 추가
mcpServers:
  - finance-api:
      type: stdio
      command: npx
      args: ["-y", "@your-org/finance-mcp-server"]
```

### 모델 변경

비용 최적화가 필요하면 서브에이전트 모델을 Haiku로 변경할 수 있습니다:

```yaml
model: haiku  # 빠르고 저렴한 분석
```

### 웹 검색 도구 추가

실시간 데이터가 필요한 서브에이전트에 웹 검색을 추가하려면 tools에 `WebSearch`를 포함하세요.

## 파일 구조

```
.claude/agents/
├── stock-analyst.md        # 오케스트레이터 A: 심층 분석
├── trading-advisor.md      # 오케스트레이터 B: 일일 매매 (GPT 연동)
├── company-profiler.md     # 서브: 기업 개요
├── financial-analyzer.md   # 서브: 재무 분석
├── industry-analyzer.md    # 서브: 산업 분석
├── momentum-analyzer.md    # 서브: 모멘텀 분석
├── risk-assessor.md        # 서브: 리스크 평가
├── daily-screener.md       # 서브: 일일 스크리닝
├── stock-comparator.md     # 서브: 비교 판정
├── README.md               # 이 파일
└── INTEGRATION_GUIDE.md    # GPT 자동매매 연동 가이드
```

## 참고

- [Anthropic 공식 서브에이전트 문서](https://code.claude.com/docs/en/sub-agents)
- [에이전트 팀 문서](https://code.claude.com/docs/en/agent-teams)
- [MCP 서버 연동](https://code.claude.com/docs/en/mcp)

## 면책 조항

이 에이전트 시스템이 생성하는 분석 리포트는 AI가 공개 정보를 기반으로 작성한 참고 자료이며,
투자 권유가 아닙니다. 모든 투자 결정은 본인의 판단과 책임 하에 이루어져야 합니다.
