# GPT 자동매매 앱 연동 가이드

> trading-advisor 에이전트 시스템과 기존 GPT OpenAPI 자동매매 앱을 연결하는 방법

## 전체 아키텍처

```
┌──────────────────────────────────────────────────────────┐
│                    사용자 (아침 루틴)                       │
│            "오늘 삼성전자, NVIDIA 보고 있어"                  │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│           🔵 trading-advisor (Claude Code)                │
│                                                          │
│   ┌─────────────┐  ┌──────────────┐                      │
│   │ daily-      │  │ 사용자 종목    │                      │
│   │ screener    │  │ 개별 분석      │                      │
│   │ (AI 추천)   │  │ (profiler +   │                      │
│   │             │  │  analyzer +   │                      │
│   │             │  │  momentum)    │                      │
│   └──────┬──────┘  └──────┬───────┘                      │
│          │                │                              │
│          └────────┬───────┘                              │
│                   ▼                                      │
│          ┌────────────────┐                              │
│          │ stock-comparator│                              │
│          │ (비교 판정)      │                              │
│          └────────┬───────┘                              │
│                   │                                      │
│                   ▼                                      │
│          📄 JSON 출력                                     │
└────────────────────────┬─────────────────────────────────┘
                         │
                    파일 저장 or API 호출
                         │
                         ▼
┌──────────────────────────────────────────────────────────┐
│           🤖 GPT 자동매매 앱 (기존 시스템)                   │
│                                                          │
│   1. JSON 파싱 → universe 종목 추출                        │
│   2. weight_pct 기반 자금 배분                              │
│   3. entry_strategy 에 따라 주문 실행                       │
│   4. stop_loss_pct 설정                                   │
│   5. excluded 종목은 매매 금지                               │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 연동 방식 3가지

### 방식 1: 파일 기반 연동 (가장 간단)

Claude Code에서 JSON 파일을 생성하고, GPT 앱이 해당 파일을 읽는 방식입니다.

**Claude Code 실행:**
```bash
# 매일 아침 실행
claude --agent trading-advisor \
  -p "오늘 내가 보는 종목: 삼성전자, NVIDIA. 코스피+나스닥 스윙 기준으로 분석해줘" \
  --output-format json > /path/to/shared/daily_universe.json
```

**GPT 자동매매 앱 코드 (Python):**
```python
import json
from datetime import date

def load_daily_universe():
    """trading-advisor가 생성한 JSON을 로드"""
    with open('/path/to/shared/daily_universe.json', 'r') as f:
        data = json.load(f)

    # 날짜 검증 — 오늘 생성된 데이터인지 확인
    if data['date'] != str(date.today()):
        raise ValueError("오래된 데이터입니다. trading-advisor를 다시 실행하세요.")

    return data

def get_trading_signals(universe_data):
    """JSON에서 매매 시그널 추출"""
    signals = []
    for stock in universe_data['universe']:
        signals.append({
            'ticker': stock['ticker'],
            'action': stock['action'],
            'weight': stock['weight_pct'] / 100,
            'stop_loss': stock['stop_loss_pct'] / 100,
            'reason': stock['reason'],
        })
    return signals

# GPT 자동매매 앱 메인 루프에서 호출
universe = load_daily_universe()
signals = get_trading_signals(universe)

# 기존 GPT 앱의 매매 로직에 전달
for signal in signals:
    execute_trade(signal)  # 기존 매매 함수
```

---

### 방식 2: Claude API 호출 연동 (자동화)

GPT 자동매매 앱에서 직접 Claude API를 호출하여 분석을 요청하는 방식입니다.

```python
import anthropic
import json

client = anthropic.Anthropic()  # ANTHROPIC_API_KEY 환경변수 사용

def get_daily_universe(user_picks: list[str], market: str, style: str):
    """Claude API로 trading-advisor 분석 요청"""

    prompt = f"""
    오늘 내가 보는 종목: {', '.join(user_picks)}
    시장: {market}
    스타일: {style}

    위 종목을 AI 추천 종목과 비교 분석하고,
    오늘의 최종 매매 유니버스를 JSON 형식으로 출력해줘.
    JSON만 출력하고 다른 설명은 하지 마.
    """

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        system=open('.claude/agents/trading-advisor.md').read(),
        messages=[{"role": "user", "content": prompt}]
    )

    # JSON 추출
    text = response.content[0].text
    json_start = text.find('{')
    json_end = text.rfind('}') + 1
    return json.loads(text[json_start:json_end])

# 사용 예시
universe = get_daily_universe(
    user_picks=["삼성전자", "NVIDIA", "카카오"],
    market="코스피 + 나스닥",
    style="스윙"
)
```

---

### 방식 3: GPT → Claude 브리지 (기존 GPT 앱 최소 수정)

기존 GPT OpenAPI 앱의 시스템 프롬프트에 Claude 분석 결과를 주입하는 방식입니다.
기존 코드를 가장 적게 수정합니다.

```python
import openai
import anthropic
import json

def get_claude_analysis(user_picks):
    """Claude에서 분석 결과 가져오기"""
    claude = anthropic.Anthropic()
    response = claude.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[{
            "role": "user",
            "content": f"종목 분석: {', '.join(user_picks)}. JSON으로 매매 유니버스 출력."
        }]
    )
    return response.content[0].text

def run_gpt_trading_with_claude_input(user_picks):
    """기존 GPT 앱에 Claude 분석 결과를 주입"""

    # Step 1: Claude 분석
    claude_analysis = get_claude_analysis(user_picks)

    # Step 2: 기존 GPT 앱의 시스템 프롬프트에 Claude 결과 추가
    enhanced_system_prompt = f"""
    {기존_GPT_시스템_프롬프트}

    [오늘의 AI 종목 분석 결과 — Claude trading-advisor 제공]
    {claude_analysis}

    위 분석 결과를 참고하여 매매를 실행하세요.
    - universe에 포함된 종목만 매매
    - weight_pct에 따라 자금 배분
    - excluded 종목은 절대 매매 금지
    - stop_loss_pct 반드시 적용
    """

    # Step 3: 기존 GPT 매매 로직 실행
    gpt = openai.OpenAI()
    response = gpt.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": enhanced_system_prompt},
            {"role": "user", "content": "오늘 매매를 시작합니다."}
        ]
    )
    return response
```

---

## GPT 앱 시스템 프롬프트 수정 가이드

기존 GPT 자동매매 앱의 시스템 프롬프트에 아래 블록을 추가합니다:

```
### Claude AI 종목 분석 연동 규칙

1. 매매 전 반드시 Claude trading-advisor의 daily_universe JSON을 확인한다.
2. JSON의 universe 배열에 포함된 종목만 매매 대상으로 한다.
3. weight_pct에 따라 총 자금을 배분한다.
4. stop_loss_pct를 모든 포지션에 적용한다.
5. excluded 배열의 종목은 어떤 이유로든 매매하지 않는다.
6. cash_weight_pct만큼의 자금은 현금으로 유지한다.
7. risk_level이 "high"이면 전체 포지션 규모를 50% 축소한다.
8. market_condition이 "bearish"이면 매수 포지션을 신중하게 진입한다.

JSON 스키마:
- universe[].ticker: 매매할 종목 코드
- universe[].action: BUY | SELL | HOLD
- universe[].weight_pct: 자금 배분 비율 (%)
- universe[].stop_loss_pct: 손절 기준 (%)
- universe[].score: AI 종합 점수 (0~100)
- universe[].grade: STRONG_BUY | BUY | HOLD | UNDERWEIGHT | AVOID
- excluded[]: 매매 금지 종목 리스트
```

---

## 일일 운영 타임라인

```
07:30  해외 시장 마감 데이터 확인
08:00  trading-advisor 실행 (사용자 종목 + AI 스크리닝)
08:30  JSON 생성 완료 → GPT 자동매매 앱에 전달
09:00  장 개시 — GPT 앱이 universe 기반으로 매매 실행
~      장 중 monitoring 항목 체크
15:30  장 마감
16:00  당일 매매 결과 기록
→      다음 날 08:00에 다시 실행
```

---

## 주의 사항

- Claude 분석은 공개 정보 기반이며, 실시간 시세 반영에는 한계가 있습니다.
- JSON 출력의 target_price는 참고용이며, 실제 매매가와 다를 수 있습니다.
- 자동매매 시스템의 오작동에 대비한 안전장치(일일 최대 손실 제한 등)를 반드시 설정하세요.
- 이 시스템은 투자 조언이 아닌 분석 도구입니다. 최종 투자 결정은 본인 책임입니다.
