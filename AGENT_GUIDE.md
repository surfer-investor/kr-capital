# KR_CAPITAL Agent Guide

## 데이터 저장소
- **GitHub**: `surfer-investor/kr-capital` / `data/portfolio.json`
- **브라우저**: localStorage `kr_capital_data`
- 브라우저는 GitHub와 자동 sync. agent가 GitHub에 push하면 브라우저 새로고침 시 반영됨.

## 데이터 구조

```json
{
  "months": {
    "2026-05": {
      "holdings": [
        { "name": "종목명", "ticker": "005930.KS", "market": "KR", "quantity": 10, "avgPrice": 70000, "currentPrice": 72000, "targetWeight": 0, "attractScore": 0 },
        { "name": "NVIDIA",  "ticker": "NVDA",      "market": "US", "quantity": 5,  "avgPrice": 100,    "currentPrice": 120 }
      ],
      "trades": [
        { "date": "2026-05-05", "name": "종목명", "ticker": "005930.KS", "market": "KR", "type": "buy", "quantity": 10, "price": 70000, "reason": "" }
      ],
      "capitalFlows": [
        { "date": "2026-05-03", "amount": 10000000, "reason": "초기 출자" },
        { "date": "2026-05-05", "amount": -100000, "reason": "회수" }
      ],
      "cashKRW": 532432,
      "cashUSD": 429.55,
      "assetHistory": [
        { "date": "2026-05-05", "totalAsset": 5340530, "note": "자동 업데이트" }
      ]
    }
  },
  "settings": { "fxRate": 1380 }
}
```

## ⚠️ 총자산 계산 (가장 자주 틀리는 부분)

```
총자산 (KRW)
  = Σ(KR holding: currentPrice × quantity)
  + Σ(US holding: currentPrice × quantity × fxRate)   ← 환율 곱하기 필수
  + cashKRW
  + cashUSD × fxRate                                   ← 환율 곱하기 필수
```

**market === "US" 인 보유종목은 currentPrice가 USD 단위**입니다. KRW 합계에 더할 때 반드시 `× fxRate`. cashUSD도 동일.

올바른 의사코드:
```python
fx = data["settings"]["fxRate"]   # 보통 1380
total = 0
for h in m["holdings"]:
    val = h["currentPrice"] * h["quantity"]
    total += val * fx if h["market"] == "US" else val
total += m.get("cashKRW", 0)
total += m.get("cashUSD", 0) * fx
```

자주 발생한 버그: US 보유종목을 KRW로 환산하지 않고 그대로 더해버림 → 총자산이 80만원~100만원 가량 적게 나옴 (US 노출 규모만큼). 자산 스냅샷을 commit하기 전 반드시 위 공식으로 검증.

## 티커 규칙
- 한국 KOSPI: `종목코드.KS` (예: `005930.KS`)
- 한국 KOSDAQ: `종목코드.KQ` (예: `263750.KQ`)
- 미국: 그냥 심볼 (예: `NVDA`, `AAPL`)
- KOSPI/KOSDAQ을 틀리면 Yahoo 시세 조회가 실패. 둘 다 시도해보고 응답 오는 쪽 사용.

## 시세 조회 (Yahoo Finance)
- 단일 종목: `https://query1.finance.yahoo.com/v8/finance/chart/<TICKER>?range=1d&interval=1d` → `chart.result[0].meta.regularMarketPrice`
- 다중 종목 batch: `https://query1.finance.yahoo.com/v8/finance/spark?symbols=A,B,C&range=1d&interval=1d` → 응답은 평면 객체 `{"<SYMBOL>":{"close":[price]}}` 포맷 (2026년 기준)
- CORS 우회 프록시(브라우저용): `api.allorigins.win/raw?url=...`. 서버 사이드는 직접 호출 가능.

## 방법 1: Browser API (Playwright/MCP)

브라우저 콘솔에서 `window.KR_CAPITAL_API` 호출. 총자산은 자동으로 올바르게 계산됨:

```javascript
KR_CAPITAL_API.recordBuy({ name, ticker, market, quantity, price, date })
KR_CAPITAL_API.recordSell({ name, ticker, market, quantity, price, date })
KR_CAPITAL_API.recordDeposit({ amount, date, reason })
KR_CAPITAL_API.recordWithdrawal({ amount, date, reason })
KR_CAPITAL_API.updatePrice(ticker, newPrice)
KR_CAPITAL_API.getPortfolio()      // 전체
KR_CAPITAL_API.getTotalAsset()     // 총자산 (FX 자동 적용)
```

## 방법 2: GitHub API (직접 JSON 수정)

### 매수 기록 시 반드시 모두 수행:
1. `holdings` 해당 종목 찾기 (ticker로 매칭)
2. 있으면: 가중평균 단가 `(oldQty*oldAvg + buyQty*buyPrice) / (oldQty+buyQty)`, quantity 증가
3. 없으면: holdings에 새 항목 push
4. `cashKRW` 차감 (KR) 또는 `cashUSD` 차감 (US)
5. `trades`에 기록 push

### 매도 기록 시:
1. `holdings`에서 quantity 차감 (0이면 제거)
2. `cashKRW` 또는 `cashUSD` 증가
3. `trades`에 기록 push

### 출자/회수:
1. `capitalFlows`에 push (출자 amount > 0, 회수 < 0)
2. `cashKRW` 동일 액수만큼 증감
3. **둘 다 반드시** — 한쪽만 수정하면 데이터 불일치

### 자산 스냅샷 (assetHistory) 기록 시:
1. **위 총자산 공식으로 계산** (US 환율 곱하기!)
2. `assetHistory`에서 같은 date 항목 제거
3. `{date, totalAsset, note}` push

브라우저에는 자가 치유 로직이 있어 오늘자 assetHistory가 잘못 계산되어 있으면 로드 시 자동 보정. 그래도 처음부터 올바르게 쓸 것.

### GitHub API 예시:
```bash
gh api repos/surfer-investor/kr-capital/contents/data/portfolio.json --jq '.content' | base64 -d > temp.json
SHA=$(gh api repos/surfer-investor/kr-capital/contents/data/portfolio.json --jq '.sha')

# JSON 수정 후
ENCODED=$(base64 -w 0 temp.json)
gh api repos/surfer-investor/kr-capital/contents/data/portfolio.json -X PUT \
  -f message="설명" -f content="$ENCODED" -f sha="$SHA"
```

## 주의사항
- **가격 검증**: 기존가 대비 ±70% 이상 변동은 프록시 오류로 간주, 무시됨
- **월 구분**: `currentMonth` 형식 `"2026-05"`. 해당 월 데이터만 조작
- **capitalFlows ↔ cash 일관성**: 자본 이동은 항상 양쪽 동시 수정
- **fxRate**: `data.settings.fxRate`에 저장. 없으면 1380 기본값 사용
