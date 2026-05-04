# KR_CAPITAL Agent Guide

## 데이터 저장소
- **GitHub**: `surfer-investor/kr-capital` / `data/portfolio.json`
- **브라우저**: localStorage `kr_capital_data`
- 브라우저에서 GitHub 자동 sync 있음. GitHub 수정 시 브라우저 새로고침하면 반영됨.

## 데이터 구조

```json
{
  "months": {
    "2026-05": {
      "holdings": [
        { "name": "종목명", "ticker": "005930.KS", "market": "KR", "quantity": 10, "avgPrice": 70000, "currentPrice": 72000, "targetWeight": 0, "attractScore": 0, "tp": 0, "sl": 0 }
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
      "assetHistory": [],
      "operationNote": "",
      "reflection": "",
      "nextMonthPlan": ""
    }
  },
  "settings": { "fxRate": 1380 }
}
```

## 티커 규칙
- 한국 KOSPI: `종목코드.KS` (예: `005930.KS`)
- 한국 KOSDAQ: `종목코드.KQ` (예: `263750.KQ`)
- 미국: 그냥 심볼 (예: `NVDA`, `AAPL`)
- **주의**: KOSPI/KOSDAQ 구분을 틀리면 시세 조회 오류 발생

## 방법 1: Browser API (Playwright MCP 사용 시)

브라우저 콘솔에서 `window.KR_CAPITAL_API` 호출:

```javascript
// 매수
KR_CAPITAL_API.recordBuy({ name: "삼성전자", ticker: "005930.KS", market: "KR", quantity: 10, price: 70000, date: "2026-05-05" })

// 매도
KR_CAPITAL_API.recordSell({ name: "삼성전자", ticker: "005930.KS", market: "KR", quantity: 5, price: 75000, date: "2026-05-05" })

// 출자 (입금)
KR_CAPITAL_API.recordDeposit({ amount: 1000000, date: "2026-05-05", reason: "추가 출자" })

// 회수 (출금)
KR_CAPITAL_API.recordWithdrawal({ amount: 100000, date: "2026-05-05", reason: "회수" })

// 조회
KR_CAPITAL_API.getPortfolio()   // 전체 포트폴리오
KR_CAPITAL_API.getHoldings()    // 보유종목
KR_CAPITAL_API.getCash()        // 현금 잔고
KR_CAPITAL_API.getTotalAsset()  // 총자산
```

## 방법 2: GitHub API (직접 JSON 수정)

### 매수 기록 시 반드시 해야 할 것:
1. `holdings`에서 해당 종목 찾기 (ticker로 매칭)
2. 있으면: 가중평균 단가 계산 `newAvg = (oldQty*oldAvg + buyQty*buyPrice) / (oldQty+buyQty)`, quantity 증가
3. 없으면: holdings에 새 항목 push
4. `cashKRW` 차감 (KR주식) 또는 `cashUSD` 차감 (US주식)
5. `trades`에 기록 push
6. **절대 빠뜨리지 말 것**: holdings 업데이트 + cash 차감 + trades 기록

### 매도 기록 시:
1. `holdings`에서 quantity 차감 (0이면 제거)
2. `cashKRW` 증가 (KR) 또는 `cashUSD` 증가 (US)
3. `trades`에 기록 push

### 출자/회수 기록 시:
1. `capitalFlows`에 push (출자: amount > 0, 회수: amount < 0)
2. `cashKRW` 조정 (출자: 증가, 회수: 차감)
3. **둘 다 반드시 해야 함** - capitalFlows만 추가하고 cash 안 바꾸면 불일치 발생

### GitHub API 예시:
```bash
# 1. 현재 데이터 읽기
gh api repos/surfer-investor/kr-capital/contents/data/portfolio.json --jq '.content' | base64 -d > temp.json
SHA=$(gh api repos/surfer-investor/kr-capital/contents/data/portfolio.json --jq '.sha')

# 2. JSON 수정 (python 등으로)

# 3. 저장
ENCODED=$(base64 -w 0 temp.json)
gh api repos/surfer-investor/kr-capital/contents/data/portfolio.json -X PUT \
  -f message="설명" -f content="$ENCODED" -f sha="$SHA"
```

## 주의사항
- **가격 검증**: 시세 업데이트 시 기존가 대비 ±70% 이상 변동은 프록시 오류로 간주되어 무시됨
- **총자산 계산**: `holdings 평가금액 합계 + cashKRW + cashUSD * fxRate`
- **월 구분**: currentMonth 형식 `"2026-05"`. 해당 월 데이터만 조작할 것
- **capitalFlows와 cash는 항상 함께 수정**: 하나만 바꾸면 데이터 불일치
