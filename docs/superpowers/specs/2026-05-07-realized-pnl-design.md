# 실현손익 추적 + 글로벌 데이터 모델 + 통화 명시

작성일: 2026-05-07
대상 파일: `index.html`

## 배경

현재 `index.html` (단일 파일 포트폴리오 트래커)에는 세 가지 구조적 문제:

1. **실현손익 데이터 손실**: 매도 시 holdings의 `quantity`만 차감 (line 1436), 전량 매도하면 holdings에서 제거되어 `avgPrice` 정보 영구 소실. trades 배열엔 매도가만 남고 매도 시점 평균단가가 안 남음.
2. **월별 데이터 모델의 결함**: `data.months[key]`가 각각 자기 `holdings/trades/cash/capitalFlows/assetHistory`를 가짐. 새 월 생성 시 자동 이월 없음 (line 929) — 사용자 수동 복사 필요. 월 경계가 인위적이고 실수 유발.
3. **통화 모호성**: 모든 trade가 `{price, market}`으로 저장. `price`가 USD인지 KRW인지 알려면 `market`을 봐야 함. 컨벤션이 코드에만 있고 데이터/입력에 없음. 결과: agent API의 `recordBuy({price})` 호출 시 market 누락하면 default 'KR' (line 2396)로 조용히 잘못 기록됨. trades 테이블 표시도 USD를 KRW처럼 `만/억`으로 포매팅 (line 1400, `fmtWon`).

## 목표 (v1)

- 매도 시점 평균단가와 실현손익을 trade에 영구 보존.
- holdings/cash/trades/capitalFlows/assetHistory를 **글로벌**로 — 월 무관 단일 데이터.
- 월별 페이지는 글로벌 데이터를 **그 달 날짜로 필터링**한 뷰.
- 모든 trade/holding에 `currency: 'USD' | 'KRW'` 명시.
- 거래내역 테이블: 단가/금액/손익을 통화별로 정확히 표시.
- 월별 카드: 한국 실현손익 / 미국 실현손익 두 줄로 표시 (FX 환산 안 함).
- 종목별 누적 실현손익 카드.
- 자산 차트에 실현/미실현 분리 표시.
- 시세 업데이트 신뢰성 개선 (CORS 프록시 다양화, 죽은 corsproxy.io 정리).

## 비목표 (v1)

- FX 효과 분리 (매수/매도 시점 환율 스냅샷). 손익은 통화별 native로 저장.
- 세금/수수료 차감 net P&L.
- FIFO 회계. 가중평균만.
- 과거 임의 시점 holdings 재구성 (replay 기능).

## 데이터 모델

### Before (현재)

```js
data = {
  months: {
    "2026-05": {
      holdings: [{name, ticker, market, quantity, avgPrice, currentPrice, ...}],
      trades: [{date, name, ticker, market, type, quantity, price}],
      cashKRW, cashUSD,
      capitalFlows: [{date, amount, reason}],
      assetHistory: [{date, totalAsset, note}],
      operationNote, reflection, nextMonthPlan
    }
  }
}
```

### After (v1)

```js
data = {
  schemaVersion: 2,
  holdings: [{
    name, ticker, market, currency,  // currency: 'USD' | 'KRW'
    quantity, avgPrice, currentPrice,
    investmentThesis, risk, targetWeight, attractScore
  }],
  trades: [{
    date, name, ticker, market, currency,
    type,           // 'buy' | 'sell'
    quantity, price,
    avgPriceAtSell, // sell only
    realizedPnL     // sell only, native currency
  }],
  cashKRW, cashUSD,
  capitalFlows: [{date, amount, currency, reason}],  // currency added
  assetHistory: [{date, totalAsset, note}],
  months: {
    "2026-05": {
      operationNote, reflection, nextMonthPlan
      // ⬆ 텍스트만
    }
  },
  lastModified
}
```

### 마이그레이션 (auto, on load)

`schemaVersion === 1 || undefined`이면:

1. `months` 키들 정렬 (오름차순). 가장 최근 월의 `holdings`가 비어있지 않으면 그것을 글로벌 holdings로. 비었으면 그 다음, 그 다음... 모두 비었으면 빈 배열.
2. cashKRW/cashUSD: 가장 최근 월의 cash 값. (legacy `cash` 필드는 `cashKRW`로.)
3. **모든** 월의 `trades` 배열을 concat → `data.trades`. 각 trade에 `currency` 필드 추가 (market === 'US' ? 'USD' : 'KRW').
4. **모든** 월의 `capitalFlows`를 concat → `data.capitalFlows`. `currency: 'KRW'` 기본값 (기존 데이터는 모두 원화 가정).
5. **모든** 월의 `assetHistory`를 concat → `data.assetHistory`.
6. 각 month 객체에서 `holdings`, `trades`, `cash*`, `capitalFlows`, `assetHistory` 삭제. 텍스트 필드만 남김.
7. holdings 각 원소에 `currency` 추가.
8. `data.schemaVersion = 2` 설정.
9. **백업**: localStorage에 `kr_capital_data_backup_v1` 키로 마이그레이션 직전 데이터 저장 (롤백용).

기존 매도 trades는 `avgPriceAtSell`/`realizedPnL` 없는 상태로 남음. 마이그레이션 시 시간순 재생으로 백필 시도:

- 빈 holdings에서 시작.
- date 오름차순으로 trades 순회.
- buy: holdings 갱신.
- sell: 매도 직전 holdings의 avgPrice를 `avgPriceAtSell`에 박고, `realizedPnL = (price - avgPriceAtSell) × quantity` 계산.
- 매도 시점 holdings에 종목이 없으면 (데이터 결손) `avgPriceAtSell = null, realizedPnL = null`. UI에서 "—" 표시.

재생 후 최종 holdings는 사용하지 않고 (실제 글로벌 holdings는 1번에서 결정) 백필만 한다.

## 코드 변경

### 데이터 접근 헬퍼 (신규)

```js
function tradesForMonth(key) { return data.trades.filter(t => t.date.startsWith(key)); }
function flowsForMonth(key) { return data.capitalFlows.filter(f => f.date.startsWith(key)); }
function assetHistoryForMonth(key) { return data.assetHistory.filter(h => h.date.startsWith(key)); }
function realizedPnLForMonth(key) {
  const tr = tradesForMonth(key).filter(t => t.type === 'sell' && t.realizedPnL != null);
  return {
    KRW: tr.filter(t => t.currency === 'KRW').reduce((s,t) => s + t.realizedPnL, 0),
    USD: tr.filter(t => t.currency === 'USD').reduce((s,t) => s + t.realizedPnL, 0),
  };
}
function realizedPnLByTicker() {
  const map = {};
  for (const t of data.trades) {
    if (t.type !== 'sell' || t.realizedPnL == null) continue;
    const k = t.ticker || t.name;
    if (!map[k]) map[k] = { name: t.name, ticker: t.ticker, currency: t.currency, KRW: 0, USD: 0 };
    map[k][t.currency] += t.realizedPnL;
  }
  return Object.values(map);
}
```

### `addTrade()` (line 1405) 변경

- `m = getMonth(currentMonth)` 제거.
- `m.holdings` → `data.holdings`, `m.trades` → `data.trades`, `m.cashKRW/USD` → `data.cashKRW/USD`.
- buy: 현재 로직 유지하되 `currency: market === 'US' ? 'USD' : 'KRW'` 박음.
- sell:
  ```js
  const existing = data.holdings.find(...);
  if (existing) {
    const avgPriceAtSell = existing.avgPrice;
    const realizedPnL = (price - avgPriceAtSell) * quantity;
    existing.quantity -= quantity;
    if (existing.quantity <= 0) data.holdings.splice(data.holdings.indexOf(existing), 1);
    data.trades.push({ date, name, ticker, market, currency, type: 'sell', quantity, price, avgPriceAtSell, realizedPnL });
  } else {
    // holdings에 없는데 매도 — 경고 + avgPriceAtSell=null
    data.trades.push({ date, name, ticker, market, currency, type: 'sell', quantity, price, avgPriceAtSell: null, realizedPnL: null });
  }
  ```
- cash 갱신은 동일.

### `addCapitalFlow()`, `addAssetHistory()`

- `m.capitalFlows.push(...)` → `data.capitalFlows.push({...})`. `addCapitalFlow`는 기본 currency를 KRW로 (cf 입력은 거의 원화) — 추후 USD 지원 시 라디오 추가.
- `addAssetHistory` → `data.assetHistory`.

### `addNewMonth()` (line 1038)

- 기존: `getMonth(input)` (holdings/trades 등 빈 객체 생성).
- 신규: `data.months[input] = { operationNote: '', reflection: '', nextMonthPlan: '' }` 만 생성.
- holdings 등은 글로벌이므로 이월 자체 개념이 없어짐.

### 렌더링

- `renderTrades(m)` → `renderTrades()`: `tradesForMonth(currentMonth)` 사용.
- `renderFlows`, `renderAssetHistory`, 자산 분석, 종목별 비중: 글로벌 holdings/cash 사용. 월 필터는 trades/flows/assetHistory에만.
- 단가 컬럼: `t.currency === 'USD' ? '$' + fmt(t.price) : fmt(t.price) + '원'`.
- 금액 컬럼: 신설 `fmtMoney(amount, currency)`:
  ```js
  function fmtMoney(amount, currency) {
    if (amount == null || isNaN(amount)) return '-';
    if (currency === 'USD') return '$' + amount.toLocaleString('en-US', {maximumFractionDigits: 2});
    // KRW: 만/억 포매팅
    const abs = Math.abs(amount);
    if (abs >= 100000000) return (amount/100000000).toFixed(1) + '억';
    if (abs >= 10000) return Math.round(amount/10000).toLocaleString('ko-KR') + '만';
    return Math.round(amount).toLocaleString('ko-KR');
  }
  ```
- 매도 row에 새 컬럼: 손익 (positive/negative class). null이면 "—".

### 월별 카드 (신규 위치)

`renderTrades` 위에 작은 요약 카드: "이달 실현손익".

```
이달 실현손익
한국:  +1,250만원  (5건)
미국:  +$430      (2건)
```

null인 trades는 카운트 제외.

### 종목별 누적 실현손익 카드 (신규)

거래 페이지 또는 새 섹션에:
- 테이블: 종목명 | 통화 | 누적 실현손익 | 매도 횟수
- `realizedPnLByTicker()` 사용.
- 정렬: 절대값 큰 순.

### 자산 차트 분리 (신규)

기존 자산 차트는 `assetHistory` (단일 KRW 합계) 기반. 변경:
- 새 시리즈 추가: "누적 실현손익" 라인 (KRW + USD×currentFx).
- 미실현 = totalAsset - 누적 실현손익 (시각적 분리는 stacked area로).
- v1에선 단순히 별도 카드/라인 하나만 추가, stacked는 v2.

### Agent API 변경

- `getPortfolio()`: `m` 의존 제거. `{holdings: data.holdings, trades: data.trades, cashKRW, cashUSD, fxRate, totalAsset, currency: 'KRW'}` 반환. 각 holding/trade에 currency 필드 포함.
- `recordBuy/Sell`:
  - `market` 누락 시 throw (default 'KR' 제거).
  - `currency` 파라미터 추가, 누락 시 `market`에서 derive (`market === 'US' ? 'USD' : 'KRW'`).
  - market과 currency 불일치 시 throw (KR+USD 또는 US+KRW 거부).
  - `recordSell` return value에 `realizedPnL` 포함.
- `addHolding`: market 필수, currency 검증 동일.

### 시세 업데이트 신뢰성

`fetchYahooPricesBatch`, `fetchYahooPrice`, 검색 함수들의 프록시 race에:

1. **죽은 `corsproxy.io` 제거** — `searchStocks` (line 1845), `searchTradeStocks` (line 1934)에 잔존. memory에 명시된 대로 삭제.
2. **`api.codetabs.com/v1/proxy?quest=` 추가** — 다른 호스팅. allorigins.win 동시 다운에 대한 보험.
3. **프록시 정의 상수화**:
   ```js
   const PROXIES = [
     u => 'https://api.allorigins.win/raw?url=' + encodeURIComponent(u),
     u => 'https://api.allorigins.win/get?url=' + encodeURIComponent(u),  // .contents 추출 필요
     u => 'https://api.codetabs.com/v1/proxy?quest=' + encodeURIComponent(u),
   ];
   ```
   각 호출 사이트가 동일 race 패턴 반복하지 않도록 헬퍼 함수로 통합.
4. **재시도 패스에서 다른 프록시부터 시도** — 1차에서 실패한 프록시는 후순위.

검증 로직 (MUTUALFUND ghost reject, 35% anomaly)은 **유지**.

## 위험과 완화

- **마이그레이션 데이터 손실**: localStorage 백업 키 보관. 마이그레이션 실패 시 fallback 경로.
- **GitHub sync 충돌**: 마이그레이션은 첫 로드 시 실행되고 saveData() 호출하면 GitHub에 즉시 push됨. 다른 기기에서 구 schema 데이터를 push하면 충돌. 완화: 로드 시 `data.schemaVersion < 2`이면 마이그레이션 후 push.
- **백필 부정확**: 과거 매도 trades 중 일부는 holdings 변경(직접 추가/삭제)으로 인해 시간순 재생이 실제와 다를 수 있음. 이 경우 `realizedPnL`을 `null`로 두고 UI에 "—"로 표시. 사용자가 수동으로 trade 수정 가능.
- **renderTrades 등 함수의 m 파라미터**: 시그니처 바꾸면 호출처 다 수정 필요. → 글로벌 사용으로 통일하여 한 번에 갈아엎음.

## 구현 순서 (위험 낮은 순)

1. **시세 업뎃 프록시 개선** (격리, 다른 코드와 무관). 검증 가능.
2. **마이그레이션 함수 + 백업** 작성. 콘솔에서 dry-run 가능하도록 분리.
3. **데이터 모델 전환**: 모든 함수의 `m.X` → `data.X` 일괄 교체. 렌더링 함수 시그니처 정리.
4. **통화 필드 + 표시 수정**: trades 테이블 currency 컬럼/표시.
5. **실현손익 계산**: addTrade의 sell 분기 + agent API.
6. **월별 요약 카드**: 이달 실현손익.
7. **종목별 누적 카드**.
8. **자산 차트 실현손익 라인**.
9. **Agent API 검증 강화**.
10. **수동 검증**: 기존 데이터 로드 → 마이그레이션 동작 확인 → 매수/매도 시뮬레이션.
