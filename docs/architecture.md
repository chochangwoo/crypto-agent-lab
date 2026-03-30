# 아키텍처 설계

## 전체 데이터 흐름

```
외부 API ──→ [데이터 수집] ──→ Supabase (market_signals)
                                    │
                                    ▼
                              [리서치 에이전트]
                              ├─ 시장 브리핑 → 텔레그램
                              └─ 전략 제안 → 티켓 생성
                                    │
                                    ▼
                              [퀀트 에이전트]
                              ├─ 백테스트 실행 (upbit-quant/backtest 연동)
                              └─ 승인 요청 → 텔레그램 + Paperclip
                                    │
                                    ▼
                              [사람] 승인/거부
                                    │
                              ┌─────┴─────┐
                              ▼           ▼
                          승인 시      거부 시
                      전략 업데이트    피드백 저장
                              │
                              ▼
                      [트레이딩 = upbit-quant]
                              │
                              ▼
                      [모니터링 에이전트]
                      ├─ 성과 추적
                      ├─ 이상 감지 → 텔레그램
                      └─ 전략 괴리 → 퀀트에 재검증 요청
```

## 에이전트 상세 설계

### 1. 데이터 수집 에이전트

별도 AI 불필요. Python 스크립트 + 스케줄러.

| 데이터 | API | 엔드포인트 | 주기 |
|--------|-----|-----------|------|
| 공포탐욕지수 | Alternative.me | `api.alternative.me/fng/` | 1시간 |
| BTC 도미넌스 | CoinGecko | `/global` | 1시간 |
| 펀딩비율 | Binance | `/fapi/v1/fundingRate` | 8시간 |
| 뉴스 헤드라인 | CryptoPanic | `/posts/` | 1시간 |
| 거래소 넷플로우 | CryptoQuant (무료 tier) | 확인 필요 | 1일 |

### 2. 리서치 에이전트

Claude API 사용. 수집된 데이터를 분석하여 인사이트 생성.

**입력**: market_signals 최근 24시간 데이터 + BTC 가격 추이
**출력**:
  - 텔레그램 시장 브리핑 (매일)
  - 전략 제안 티켓 (조건 충족 시)

**전략 제안 트리거 조건 예시**:
  - 공포탐욕지수 < 20 (극단적 공포) 3일 연속
  - 펀딩비율 음전환 (숏 과열 → 반등 가능성)
  - 뉴스 감성 급변 (±0.5 이상 변화)

### 3. 퀀트 에이전트

Claude API + upbit-quant/backtest 엔진 연동.

**입력**: 리서치 에이전트의 전략 제안 티켓
**처리**:
  1. 제안을 백테스트 코드로 변환
  2. upbit-quant/backtest/engine.py로 실행
  3. 기존 전략 vs 변형 전략 비교표 생성
  4. 통계적 유의성 검증 (몬테카를로, 부트스트랩)
**출력**: 승인 요청 (비교표 + 리스크 분석 포함)

### 4. 트레이딩 에이전트

= upbit-quant 기존 봇. 이 프로젝트에서 직접 수정하지 않음.
승인된 전략 변경은 upbit-quant repo에 PR로 반영.

### 5. 모니터링 에이전트

별도 AI 불필요 (대부분). 규칙 기반 감시.

| 감시 | 기준 | 행동 |
|------|------|------|
| 일일 손실 | > -5% | 긴급 알림 |
| MDD | > -15% | 전략 중단 승인 요청 |
| 전략 괴리 | 실전 vs 백테스트 > 10%p | 퀀트에 재검증 요청 |
| 데이터 수집 실패 | 3회 연속 | 수집기 재시작 |

## upbit-quant 연동 방식

이 프로젝트는 upbit-quant를 **직접 수정하지 않는다**.

연동 방식:
1. **Supabase 공유**: 동일 Supabase 프로젝트의 다른 테이블 사용
2. **텔레그램 공유**: 동일 봇, 다른 메시지 포맷 (접두어로 구분)
3. **백테스트 엔진**: upbit-quant/backtest를 패키지로 import하여 사용
4. **전략 변경 반영**: 승인 시 upbit-quant repo에 PR 생성 (GitHub API)

## Supabase 신규 테이블

```sql
-- 외부 시장 데이터
CREATE TABLE market_signals (
    id          BIGSERIAL PRIMARY KEY,
    timestamp   TIMESTAMPTZ NOT NULL,
    signal_type TEXT NOT NULL,          -- 'fear_greed', 'funding_rate', etc.
    value       NUMERIC,
    metadata    JSONB,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 전략 제안 (리서치 → 퀀트)
CREATE TABLE strategy_proposals (
    id          BIGSERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    description TEXT,
    trigger     TEXT,                   -- 제안 트리거 사유
    status      TEXT DEFAULT 'pending', -- pending/testing/approved/rejected
    backtest_result JSONB,              -- 백테스트 결과
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    reviewed_at TIMESTAMPTZ
);

-- 에이전트 활동 로그
CREATE TABLE agent_logs (
    id          BIGSERIAL PRIMARY KEY,
    agent_name  TEXT NOT NULL,
    action      TEXT NOT NULL,
    details     JSONB,
    created_at  TIMESTAMPTZ DEFAULT NOW()
);
```
