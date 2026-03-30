# Crypto Agent Lab - 프로젝트 규칙

## 프로젝트 개요
- **목적**: 크립토 리서치 + 트레이딩 멀티 에이전트 파이프라인
- **언어**: Python 3.13
- **OS**: Windows 11 (개발)
- **관계**: upbit-quant(실거래 봇)와 별도 프로젝트, Supabase/텔레그램 공유

## 핵심 원칙
1. **upbit-quant를 직접 수정하지 않는다** — 연동은 Supabase/API를 통해
2. **사람의 승인 없이 전략을 변경하지 않는다** — 거버넌스 필수
3. **Phase별 점진적 구현** — Phase 1(데이터)부터 순서대로

## 기술 스택
| 역할 | 도구 |
|------|------|
| AI | Claude API (anthropic SDK) |
| 데이터베이스 | Supabase (upbit-quant와 동일 인스턴스) |
| 알림 | 텔레그램 봇 (upbit-quant와 동일 봇) |
| 스케줄링 | schedule 라이브러리 |
| 로깅 | loguru |
| 오케스트레이션 | Paperclip (Phase 3~) |

## 폴더 구조
```
crypto-agent-lab/
├── data/
│   ├── collectors/           # 외부 데이터 수집기
│   │   ├── fear_greed.py
│   │   ├── funding_rate.py
│   │   ├── btc_dominance.py
│   │   └── news_sentiment.py
│   └── scheduler.py          # 수집 스케줄러
├── agents/
│   ├── research/             # 리서치 에이전트
│   │   ├── daily_briefing.py
│   │   ├── correlation_scanner.py
│   │   └── strategy_proposer.py
│   ├── quant/                # 퀀트 에이전트
│   │   ├── backtest_runner.py
│   │   └── approval_request.py
│   └── monitor/              # 모니터링 에이전트
│       ├── performance_tracker.py
│       └── anomaly_detector.py
├── config/
│   └── settings.yaml
├── docs/
│   └── architecture.md
├── tests/
├── requirements.txt
├── .env
├── .env.example
├── README.md
└── CLAUDE.md
```

## 코딩 규칙
1. 모든 주석 및 docstring은 한국어로 작성
2. API 키는 반드시 .env 파일에서만 관리
3. 설정값은 config/settings.yaml에서 관리
4. loguru로 로그 출력
5. 텔레그램 메시지는 접두어로 구분: `[리서치]`, `[퀀트]`, `[모니터]`

## 개발 현황
- [x] 프로젝트 구조 설계
- [x] 아키텍처 문서 작성
- [ ] Phase 1: 데이터 수집기 구현
- [ ] Phase 2: 리서치 에이전트 구현
- [ ] Phase 3: 퀀트 에이전트 + Paperclip 도입
- [ ] Phase 4: 모니터링 + upbit-quant 연동
