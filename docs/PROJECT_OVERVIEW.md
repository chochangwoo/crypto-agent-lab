# Crypto Agent Lab - 프로젝트 개요

## 소개

외부 데이터 수집 → AI 리서치 → 자동 백테스트 → 사람 승인 → 실거래 반영까지의
전체 파이프라인을 **멀티 에이전트**로 자동화하는 시스템.

- **상태**: 아키텍처 설계 완료, 코드 구현 미시작
- **기술 스택**: Python 3.13, Claude API (Anthropic SDK), Supabase, Telegram
- **연동**: upbit-quant 실거래 봇과 Supabase/Telegram 공유

---

## 핵심 파이프라인 (5단계)

```
[1] 데이터 수집 에이전트 (매 1시간)
     공포탐욕지수, 펀딩비율, BTC 도미넌스, 뉴스 감성
              ↓
[2] 리서치 에이전트 (매일 09:00, Claude API)
     시장 브리핑, 상관 분석, 전략 아이디어 제안
              ↓
[3] 퀀트 에이전트 (티켓 수신/주 1회, Claude API)
     백테스트 자동 실행, 통계 검증, 승인 요청
              ↓
[4] 사람의 승인/거부 (Telegram)
              ↓
[5] 트레이딩 에이전트 → upbit-quant 전략 업데이트
              ↓
[6] 모니터링 에이전트 (30분마다)
     성과 추적, 이상 감지, 전략 괴리 알림
```

---

## 프로젝트 구조

```
crypto-agent-lab/
├── data/
│   ├── collectors/              # 외부 데이터 수집기
│   │   ├── fear_greed.py        # 공포탐욕지수 (Alternative.me)
│   │   ├── funding_rate.py      # 펀딩비율 (Binance)
│   │   ├── btc_dominance.py     # BTC 도미넌스 (CoinGecko)
│   │   └── news_sentiment.py    # 뉴스 감성 (CryptoPanic)
│   └── scheduler.py             # 수집 스케줄러
├── agents/
│   ├── research/                # 리서치 에이전트
│   │   ├── daily_briefing.py    # 일일 시장 브리핑
│   │   ├── correlation_scanner.py
│   │   └── strategy_proposer.py # 전략 아이디어 생성
│   ├── quant/                   # 퀀트 에이전트
│   │   ├── backtest_runner.py   # 자동 백테스트
│   │   └── approval_request.py  # 승인 플로우
│   └── monitor/                 # 모니터링 에이전트
│       ├── performance_tracker.py
│       └── anomaly_detector.py  # 이상 감지
├── config/settings.yaml
├── docs/architecture.md         # 상세 아키텍처
└── README.md
```

---

## 데이터 소스

| 데이터 | API | 수집 주기 |
|--------|-----|---------|
| 공포탐욕지수 | Alternative.me | 1시간 |
| BTC 도미넌스 | CoinGecko | 1시간 |
| 펀딩비율 | Binance | 8시간 |
| 뉴스 헤드라인 | CryptoPanic | 1시간 |
| 거래소 넷플로우 | CryptoQuant | 1일 |

### 전략 제안 트리거 예시
- 공포탐욕지수 < 20 (극단적 공포) 3일 연속
- 펀딩비율 음전환 (숏 과열 → 반등 가능성)
- 뉴스 감성 급변 (±0.5 이상)

---

## upbit-quant 연동 방식

기존 upbit-quant를 **직접 수정하지 않는** 비침투적 연동:

1. **Supabase 공유**: 동일 인스턴스, 별도 테이블 (market_signals, strategy_proposals, agent_logs)
2. **Telegram 공유**: 동일 봇, 메시지 접두어로 구분 ([리서치], [퀀트], [모니터])
3. **백테스트 엔진**: upbit-quant/backtest를 import하여 사용
4. **전략 변경**: 승인 시 upbit-quant repo에 PR 생성 (GitHub API)

---

## Supabase 신규 테이블

```sql
-- 외부 시장 데이터
market_signals (id, timestamp, signal_type, value, metadata)

-- 전략 제안 (리서치 → 퀀트)
strategy_proposals (id, title, description, trigger, status, backtest_result, created_at, reviewed_at)

-- 에이전트 활동 로그
agent_logs (id, agent_name, action, details, created_at)
```

---

## 개발 로드맵

### Phase 1: 데이터 기반 구축
- [ ] 공포탐욕/펀딩비율/BTC도미넌스/뉴스 수집기
- [ ] Supabase market_signals 테이블
- [ ] 수집 스케줄러

### Phase 2: 리서치 자동화
- [ ] 일일 시장 브리핑 (Claude API)
- [ ] 외부 데이터 ↔ 수익률 상관 분석
- [ ] 전략 아이디어 자동 제안

### Phase 3: 퀀트 자동화
- [ ] 백테스트 자동 실행
- [ ] 승인 요청 플로우 (Telegram)
- [ ] 전략 자동 배포

### Phase 4: 모니터링 + 통합
- [ ] 실시간 성과 추적
- [ ] 이상 감지 (MDD, 슬리피지)
- [ ] 전략 괴리 감지 (실전 vs 백테스트)

---

## 기술 스택

| 역할 | 기술 |
|------|------|
| 언어 | Python 3.13 |
| AI/LLM | Claude API (Anthropic SDK) |
| DB | Supabase (PostgreSQL) |
| 알림 | Telegram Bot API |
| 스케줄링 | schedule |
| 로깅 | loguru |

### 예상 월 비용
- 데이터 수집: $0 (무료 API)
- Claude API (리서치+퀀트): $15~45
- 합계: **$15~45/월**

---

## 개발 규칙

1. 모든 주석/docstring은 한국어
2. API 키는 .env 파일에서만 관리
3. 설정값은 config/settings.yaml
4. Telegram 메시지는 접두어 구분: [리서치], [퀀트], [모니터]
5. **upbit-quant를 직접 수정하지 않는다** (Supabase/API로 연동)
6. **사람의 승인 없이 전략 변경 불가**
7. Phase별 점진적 구현

---

## 현재 상태

- ✅ 프로젝트 구조 및 아키텍처 설계 완료
- ✅ 개발 규칙/가이드라인 정의
- ❌ 모든 Python 코드 미구현
- ❌ requirements.txt 미정의

---

## 관련 프로젝트

| 프로젝트 | 역할 |
|----------|------|
| [upbit-quant](../upbit-quant/) | 실거래 자동매매 봇 (Railway 24/7) |
| [salkkamalka-backtest](../salkkamalka-backtest/) | 백테스트 웹 플랫폼 |
