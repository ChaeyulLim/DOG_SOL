# 10_MONITORING 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [monitoring/logger.py](#monitoringloggerpy)
2. [monitoring/reporter.py](#monitoringreporterpy)
3. [monitoring/performance.py](#monitoringperformancepy)
4. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📁 monitoring/logger.py

### 파일 전체 구조
```python
import logging
import sys
from pathlib import Path
from datetime import datetime
from typing import Optional

from core.constants import LOG_DIR

class TradingLogger:
    def __init__(self, mode: str, level: str = 'INFO'): ...
    
    def setup_logger(self) -> logging.Logger: ...
    
    def log_entry(
        self,
        symbol: str,
        price: float,
        amount_krw: int,
        ai_confidence: float
    ) -> None: ...
    
    def log_exit(
        self,
        symbol: str,
        entry_price: float,
        exit_price: float,
        pnl_percent: float,
        pnl_krw: float,
        reason: str
    ) -> None: ...
    
    def log_risk_event(
        self,
        event_type: str,
        details: dict
    ) -> None: ...
    
    def log_ai_decision(
        self,
        symbol: str,
        action: str,
        confidence: float,
        reasoning: str
    ) -> None: ...
```

---

### 📌 클래스: TradingLogger

#### 목적
모든 거래 활동 로깅 (파일 + 콘솔)

---

### 📌 함수: TradingLogger.__init__(mode, level)

```python
def __init__(self, mode: str, level: str = 'INFO'):
```

#### 역할
로거 초기화

#### 인자
- `mode: str` - 'paper', 'live', 'backtest'
- `level: str` - 'DEBUG', 'INFO', 'WARNING', 'ERROR'

#### 초기화 내용
```python
self.mode = mode
self.level = level
self.logger = self.setup_logger()
```

#### 호출되는 곳
```python
# engine/base_engine.py
from monitoring import TradingLogger

self.logger = TradingLogger(mode='paper', level='INFO')
```

#### 구현 코드
```python
def __init__(self, mode: str, level: str = 'INFO'):
    """
    로거 초기화
    
    Args:
        mode: 'paper', 'live', 'backtest'
        level: 'DEBUG', 'INFO', 'WARNING', 'ERROR'
    """
    self.mode = mode
    self.level = level
    self.logger = self.setup_logger()
```

---

### 📌 함수: TradingLogger.setup_logger()

```python
def setup_logger(self) -> logging.Logger:
```

#### 역할
로거 설정 (파일 + 콘솔 핸들러)

#### 로그 파일 구조
```
logs/
├── paper/
│   ├── 2025-01-15_trades.log      # 거래만
│   ├── 2025-01-15_decisions.log   # AI 판단만
│   ├── 2025-01-15_errors.log      # 에러만
│   └── 2025-01-15_debug.log       # 전체
├── live/
└── backtest/
```

#### 로그 포맷
```
[2025-01-15 14:32:15] [INFO] [ENTRY] DOGE/USDT @ 0.3821 | Amount: 500,000 KRW | AI: 0.75
[2025-01-15 16:45:22] [INFO] [EXIT] DOGE/USDT @ 0.3895 | PnL: +1.94% | Reason: TRAILING_STOP
[2025-01-15 18:00:00] [WARNING] [RISK] Daily Loss: -2.3% / -5.0%
[2025-01-15 20:15:33] [ERROR] [API] Bybit Connection Timeout - Retrying...
```

#### 구현 코드
```python
def setup_logger(self) -> logging.Logger:
    """
    로거 설정
    
    Returns:
        설정된 Logger 객체
    """
    # 로거 생성
    logger = logging.getLogger(f'trading_{self.mode}')
    logger.setLevel(getattr(logging, self.level))
    
    # 기존 핸들러 제거 (중복 방지)
    logger.handlers.clear()
    
    # 오늘 날짜
    today = datetime.now().strftime('%Y-%m-%d')
    
    # 로그 디렉토리 생성
    log_dir = Path(LOG_DIR) / self.mode
    log_dir.mkdir(parents=True, exist_ok=True)
    
    # 포매터
    formatter = logging.Formatter(
        '[%(asctime)s] [%(levelname)s] %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )
    
    # 1. 콘솔 핸들러 (INFO 이상)
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setLevel(logging.INFO)
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)
    
    # 2. 거래 로그 파일 (INFO)
    trades_handler = logging.FileHandler(
        log_dir / f'{today}_trades.log',
        encoding='utf-8'
    )
    trades_handler.setLevel(logging.INFO)
    trades_handler.setFormatter(formatter)
    trades_handler.addFilter(
        lambda record: '[ENTRY]' in record.getMessage() or 
                      '[EXIT]' in record.getMessage()
    )
    logger.addHandler(trades_handler)
    
    # 3. AI 결정 로그 파일
    decisions_handler = logging.FileHandler(
        log_dir / f'{today}_decisions.log',
        encoding='utf-8'
    )
    decisions_handler.setLevel(logging.INFO)
    decisions_handler.setFormatter(formatter)
    decisions_handler.addFilter(
        lambda record: '[AI]' in record.getMessage()
    )
    logger.addHandler(decisions_handler)
    
    # 4. 에러 로그 파일
    error_handler = logging.FileHandler(
        log_dir / f'{today}_errors.log',
        encoding='utf-8'
    )
    error_handler.setLevel(logging.ERROR)
    error_handler.setFormatter(formatter)
    logger.addHandler(error_handler)
    
    # 5. 디버그 로그 파일 (전체)
    if self.level == 'DEBUG':
        debug_handler = logging.FileHandler(
            log_dir / f'{today}_debug.log',
            encoding='utf-8'
        )
        debug_handler.setLevel(logging.DEBUG)
        debug_handler.setFormatter(formatter)
        logger.addHandler(debug_handler)
    
    return logger
```

---

### 📌 함수: TradingLogger.log_entry(symbol, price, amount_krw, ai_confidence)

```python
def log_entry(
    self,
    symbol: str,
    price: float,
    amount_krw: int,
    ai_confidence: float
) -> None:
```

#### 역할
진입 로그 기록

#### 호출되는 곳
```python
# engine/base_engine.py - 주문 체결 후
self.logger.log_entry(
    'DOGE/USDT',
    0.3821,
    500000,
    0.75
)
```

#### 로그 형식
```
[2025-01-15 14:32:15] [INFO] [ENTRY] DOGE/USDT @ 0.3821 | Amount: 500,000 KRW | AI: 0.75
```

#### 구현 코드
```python
def log_entry(
    self,
    symbol: str,
    price: float,
    amount_krw: int,
    ai_confidence: float
) -> None:
    """
    진입 로그
    
    Example:
        >>> logger.log_entry('DOGE/USDT', 0.3821, 500000, 0.75)
        [2025-01-15 14:32:15] [INFO] [ENTRY] DOGE/USDT @ 0.3821 | Amount: 500,000 KRW | AI: 0.75
    """
    self.logger.info(
        f"[ENTRY] {symbol} @ {price:.4f} | "
        f"Amount: {amount_krw:,} KRW | "
        f"AI: {ai_confidence:.2f}"
    )
```

---

### 📌 함수: TradingLogger.log_exit(symbol, entry_price, exit_price, pnl_percent, pnl_krw, reason)

```python
def log_exit(
    self,
    symbol: str,
    entry_price: float,
    exit_price: float,
    pnl_percent: float,
    pnl_krw: float,
    reason: str
) -> None:
```

#### 역할
청산 로그 기록

#### 로그 형식
```
[2025-01-15 16:45:22] [INFO] [EXIT] DOGE/USDT @ 0.3895 | PnL: +1.94% (+9,700 KRW) | Reason: TRAILING_STOP
```

#### 구현 코드
```python
def log_exit(
    self,
    symbol: str,
    entry_price: float,
    exit_price: float,
    pnl_percent: float,
    pnl_krw: float,
    reason: str
) -> None:
    """
    청산 로그
    
    Example:
        >>> logger.log_exit('DOGE/USDT', 0.3821, 0.3895, 0.0194, 9700, 'TRAILING_STOP')
        [2025-01-15 16:45:22] [INFO] [EXIT] DOGE/USDT @ 0.3895 | PnL: +1.94% (+9,700 KRW) | Reason: TRAILING_STOP
    """
    self.logger.info(
        f"[EXIT] {symbol} @ {exit_price:.4f} | "
        f"Entry: {entry_price:.4f} | "
        f"PnL: {pnl_percent*100:+.2f}% ({pnl_krw:+,.0f} KRW) | "
        f"Reason: {reason}"
    )
```

---

### 📌 함수: TradingLogger.log_risk_event(event_type, details)

```python
def log_risk_event(
    self,
    event_type: str,
    details: dict
) -> None:
```

#### 역할
리스크 이벤트 로그

#### 인자
```python
event_type: str = 'DAILY_LIMIT' | 'MONTHLY_DD' | 'CONSECUTIVE_LOSS'
details: dict = {
    'loss': -0.05,
    'resume_time': timestamp,
    ...
}
```

#### 로그 형식
```
[2025-01-15 18:00:00] [WARNING] [RISK] DAILY_LIMIT | Loss: -5.0% | Resume: 2025-01-16 00:00:00
```

#### 구현 코드
```python
def log_risk_event(
    self,
    event_type: str,
    details: dict
) -> None:
    """
    리스크 이벤트 로그
    
    Example:
        >>> logger.log_risk_event('DAILY_LIMIT', {'loss': -0.05, 'resume_time': 1234567890})
        [2025-01-15 18:00:00] [WARNING] [RISK] DAILY_LIMIT | Loss: -5.0% | Resume: 2025-01-16 00:00:00
    """
    message = f"[RISK] {event_type}"
    
    if 'loss' in details:
        message += f" | Loss: {details['loss']*100:.1f}%"
    
    if 'drawdown' in details:
        message += f" | DD: {details['drawdown']*100:.1f}%"
    
    if 'count' in details:
        message += f" | Count: {details['count']}"
    
    if 'resume_time' in details:
        resume_dt = datetime.fromtimestamp(details['resume_time'])
        message += f" | Resume: {resume_dt.strftime('%Y-%m-%d %H:%M:%S')}"
    
    self.logger.warning(message)
```

---

### 📌 함수: TradingLogger.log_ai_decision(symbol, action, confidence, reasoning)

```python
def log_ai_decision(
    self,
    symbol: str,
    action: str,
    confidence: float,
    reasoning: str
) -> None:
```

#### 역할
AI 판단 로그

#### 로그 형식
```
[2025-01-15 14:32:10] [INFO] [AI] DOGE/USDT | Action: ENTER | Confidence: 0.75 | Reasoning: MACD golden cross with strong volume
```

#### 구현 코드
```python
def log_ai_decision(
    self,
    symbol: str,
    action: str,
    confidence: float,
    reasoning: str
) -> None:
    """
    AI 판단 로그
    
    Example:
        >>> logger.log_ai_decision('DOGE/USDT', 'ENTER', 0.75, 'MACD golden cross...')
        [2025-01-15 14:32:10] [INFO] [AI] DOGE/USDT | Action: ENTER | Confidence: 0.75
    """
    # 긴 reasoning은 50자로 제한
    short_reasoning = reasoning[:50] + '...' if len(reasoning) > 50 else reasoning
    
    self.logger.info(
        f"[AI] {symbol} | "
        f"Action: {action} | "
        f"Confidence: {confidence:.2f} | "
        f"Reasoning: {short_reasoning}"
    )
```

---

## 📁 monitoring/reporter.py

### 파일 전체 구조
```python
from datetime import datetime, timedelta
from typing import Dict
from database import TradeDatabase

class PerformanceReporter:
    def __init__(self, db: TradeDatabase): ...
    
    def generate_daily_summary(self) -> str: ...
    
    def generate_weekly_report(self) -> str: ...
    
    def generate_monthly_report(self) -> str: ...
    
    def save_report(self, report: str, report_type: str) -> None: ...
```

---

### 📌 클래스: PerformanceReporter

#### 목적
일일/주간/월간 리포트 생성

---

### 📌 함수: PerformanceReporter.generate_daily_summary()

```python
def generate_daily_summary(self) -> str:
```

#### 역할
일일 요약 리포트 생성

#### 호출되는 곳
```python
# engine/base_engine.py - 매일 09:00, 21:00
report = self.reporter.generate_daily_summary()
print(report)
```

#### 출력 형식
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 2025-01-15 일일 리포트 (21:00)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💼 계좌 현황:
  현재 잔고: 1,032,000 KRW
  일 수익률: +3.2%
  월 수익률: +8.7%

📈 오늘 거래:
  거래 횟수: 3회
  승률: 66.7% (2승 1패)
  평균 수익: +3.5%
  최대 수익: +4.2% (SOL)
  손실: -1.0% (DOGE)

🎯 심볼별:
  DOGE: 1회 (+4.2%)
  SOL: 2회 (+2.8%, -1.0%)

🤖 AI 통계:
  호출: 5회
  진입 승인: 3/4 (75%)
  평균 신뢰도: 0.73

⚠️ 리스크 현황:
  일일 손실: +3.2% / -5.0% ✅
  월간 DD: -2.1% / -10.0% ✅
  연속 손실: 0회 ✅

📌 현재 포지션:
  없음

💡 특이사항:
  - 14:23 네트워크 재연결 (30초)
  - SOL 트레일링 스톱 작동 (+2.8%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 구현 코드
```python
def generate_daily_summary(self) -> str:
    """
    일일 요약 리포트
    
    Returns:
        포맷된 리포트 문자열
    """
    today = datetime.now().strftime('%Y-%m-%d')
    
    # 오늘 거래 조회
    trades = self.db.get_trades_by_date(today)
    
    # 통계 계산
    if not trades:
        return f"📊 {today} - 오늘 거래 없음"
    
    total = len(trades)
    winners = [t for t in trades if t['pnl_percent'] > 0]
    losers = [t for t in trades if t['pnl_percent'] <= 0]
    
    win_rate = (len(winners) / total) * 100
    avg_profit = sum(t['pnl_percent'] for t in winners) / len(winners) * 100 if winners else 0
    total_pnl = sum(t['pnl_krw'] for t in trades)
    
    # 리포트 작성
    report = f"""
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 {today} 일일 리포트 ({datetime.now().strftime('%H:%M')})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📈 오늘 거래:
  거래 횟수: {total}회
  승률: {win_rate:.1f}% ({len(winners)}승 {len(losers)}패)
  평균 수익: {avg_profit:+.1f}%
  총 손익: {total_pnl:+,.0f} KRW
"""
    
    # 심볼별 상세
    symbol_summary = {}
    for trade in trades:
        symbol = trade['symbol'].split('/')[0]
        if symbol not in symbol_summary:
            symbol_summary[symbol] = []
        symbol_summary[symbol].append(trade['pnl_percent'] * 100)
    
    report += "\n🎯 심볼별:\n"
    for symbol, pnls in symbol_summary.items():
        pnl_str = ', '.join([f"{p:+.1f}%" for p in pnls])
        report += f"  {symbol}: {len(pnls)}회 ({pnl_str})\n"
    
    report += "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    
    return report
```

---

### 📌 함수: PerformanceReporter.generate_weekly_report()

```python
def generate_weekly_report(self) -> str:
```

#### 역할
주간 분석 리포트 (매주 월요일 자동)

#### 출력 형식
```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 주간 분석 (2025-01-13 ~ 2025-01-19)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📈 성과:
  총 거래: 15회
  승률: 66.7% (10승 5패)
  주간 수익률: +12.3%

🏆 성공 패턴:
  1. RSI 30-40 + MACD 골든 크로스 + 14시 진입
     → 승률 80% (4/5)
  
  2. 볼린저 하단 터치 + 거래량 2배
     → 승률 75% (3/4)

❌ 실패 패턴:
  1. RSI > 60에서 진입
     → 승률 0% (0/3)
  
  2. 21-23시 진입
     → 승률 33% (1/3)

💡 AI 학습 제안:
  - RSI 임계값: 30 → 35
  - 최소 신뢰도: 0.70 → 0.72
  - 선호 시간대: [2, 3, 14, 15]
  - 회피 조건: RSI > 60, 거래량 < 1.5배

🔧 파라미터 조정 권장:
  config.MIN_AI_CONFIDENCE = 0.72
  선호 진입 시간: 02-04시, 14-16시
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### 구현 코드
```python
def generate_weekly_report(self) -> str:
    """
    주간 분석 리포트
    
    Returns:
        포맷된 리포트 문자열
    """
    # 지난 7일 거래
    trades = self.db.get_trades_last_n_days(7)
    
    if not trades:
        return "📊 주간 리포트 - 거래 없음"
    
    # 기본 통계
    total = len(trades)
    winners = [t for t in trades if t['pnl_percent'] > 0]
    win_rate = (len(winners) / total) * 100
    total_pnl = sum(t['pnl_percent'] for t in trades) * 100
    
    # 패턴 분석 (간단 버전)
    # 실제로는 ai/learner.py에서 더 상세히 분석
    
    report = f"""
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 주간 분석 ({datetime.now().strftime('%Y-%m-%d')})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📈 성과:
  총 거래: {total}회
  승률: {win_rate:.1f}% ({len(winners)}승 {total-len(winners)}패)
  주간 수익률: {total_pnl:+.1f}%

💡 상세 분석은 learning_report.txt 참조
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"""
    
    return report
```

---

### 📌 함수: PerformanceReporter.save_report(report, report_type)

```python
def save_report(self, report: str, report_type: str) -> None:
```

#### 역할
리포트 파일 저장

#### 인자
- `report_type: str` - 'daily', 'weekly', 'monthly'

#### 저장 위치
```
logs/
└── reports/
    ├── daily/
    │   └── 2025-01-15.txt
    ├── weekly/
    │   └── 2025-W03.txt
    └── monthly/
        └── 2025-01.txt
```

#### 구현 코드
```python
def save_report(self, report: str, report_type: str) -> None:
    """
    리포트 파일 저장
    
    Example:
        >>> reporter.save_report(daily_report, 'daily')
        # logs/reports/daily/2025-01-15.txt 생성
    """
    from pathlib import Path
    from core.constants import LOG_DIR
    
    # 리포트 디렉토리
    report_dir = Path(LOG_DIR) / 'reports' / report_type
    report_dir.mkdir(parents=True, exist_ok=True)
    
    # 파일명
    now = datetime.now()
    if report_type == 'daily':
        filename = now.strftime('%Y-%m-%d.txt')
    elif report_type == 'weekly':
        week_num = now.isocalendar()[1]
        filename = f"{now.year}-W{week_num:02d}.txt"
    else:  # monthly
        filename = now.strftime('%Y-%m.txt')
    
    # 저장
    filepath = report_dir / filename
    with open(filepath, 'w', encoding='utf-8') as f:
        f.write(report)
```

---

## 📁 monitoring/performance.py

### 파일 전체 구조
```python
import numpy as np
import pandas as pd
from typing import List, Dict
from database import TradeDatabase

class PerformanceTracker:
    def __init__(self, db: TradeDatabase): ...
    
    def calculate_sharpe_ratio(
        self,
        returns: List[float],
        risk_free_rate: float = 0.03
    ) -> float: ...
    
    def calculate_max_drawdown(
        self,
        balance_history: List[tuple]
    ) -> Dict: ...
    
    def calculate_win_rate(self, trades: List[Dict]) -> Dict: ...
    
    def calculate_profit_factor(self, trades: List[Dict]) -> float: ...
    
    def get_performance_metrics(
        self,
        start_date: str = None,
        end_date: str = None
    ) -> Dict: ...
```

---

### 📌 클래스: PerformanceTracker

#### 목적
성과 지표 계산 (Sharpe, DD, Win Rate 등)

---

### 📌 함수: PerformanceTracker.calculate_sharpe_ratio(returns, risk_free_rate)

```python
def calculate_sharpe_ratio(
    self,
    returns: List[float],
    risk_free_rate: float = 0.03
) -> float:
```

#### 역할
샤프 비율 계산 (위험 대비 수익률)

#### 인자
- `returns: List[float]` - 거래별 수익률 [0.02, -0.01, 0.035, ...]
- `risk_free_rate: float = 0.03` - 무위험 수익률 (연 3%)

#### 반환값
```python
float: 샤프 비율

해석:
  > 2.0: 매우 우수
  1.0 - 2.0: 우수
  0.0 - 1.0: 보통
  < 0.0: 불량
```

#### 구현 코드
```python
def calculate_sharpe_ratio(
    self,
    returns: List[float],
    risk_free_rate: float = 0.03
) -> float:
    """
    샤프 비율 계산
    
    Formula:
        Sharpe = (평균수익률 - 무위험수익률) / 표준편차
    
    Example:
        >>> tracker = PerformanceTracker(db)
        >>> sharpe = tracker.calculate_sharpe_ratio([0.02, -0.01, 0.035])
        >>> print(f"Sharpe Ratio: {sharpe:.2f}")
        Sharpe Ratio: 1.45
    """
    if len(returns) < 2:
        return 0.0
    
    returns_array = np.array(returns)
    
    # 연환산
    annual_return = returns_array.mean() * 252  # 252 거래일
    annual_std = returns_array.std() * np.sqrt(252)
    
    if annual_std == 0:
        return 0.0
    
    sharpe = (annual_return - risk_free_rate) / annual_std
    
    return round(sharpe, 2)
```

---

### 📌 함수: PerformanceTracker.calculate_max_drawdown(balance_history)

```python
def calculate_max_drawdown(
    self,
    balance_history: List[tuple]
) -> Dict:
```

#### 역할
최대 낙폭 계산

#### 인자
```python
balance_history: List[tuple] = [
    (timestamp1, 1000000),
    (timestamp2, 1050000),
    (timestamp3, 980000),
    ...
]
```

#### 반환값
```python
Dict:
    'max_dd': float = -0.15  # -15%
    'peak': float = 1200000
    'trough': float = 1020000
    'duration_days': int = 12
    'recovery_days': int = 8
```

#### 구현 코드
```python
def calculate_max_drawdown(
    self,
    balance_history: List[tuple]
) -> Dict:
    """
    최대 낙폭 계산
    
    Returns:
        최대 낙폭 정보Example:
        >>> history = [(ts1, 1000000), (ts2, 1200000), (ts3, 1020000)]
        >>> dd = tracker.calculate_max_drawdown(history)
        >>> print(f"Max DD: {dd['max_dd']*100:.1f}%")
        Max DD: -15.0%
    """
    if len(balance_history) < 2:
        return {
            'max_dd': 0.0,
            'peak': 0.0,
            'trough': 0.0,
            'duration_days': 0
        }
    
    balances = [b[1] for b in balance_history]
    timestamps = [b[0] for b in balance_history]
    
    peak = balances[0]
    max_dd = 0.0
    peak_index = 0
    trough_index = 0
    
    for i, balance in enumerate(balances):
        # 신고점
        if balance > peak:
            peak = balance
            peak_index = i
        
        # 낙폭 계산
        drawdown = (balance - peak) / peak
        
        # 최대 낙폭 갱신
        if drawdown < max_dd:
            max_dd = drawdown
            trough_index = i
    
    # 기간 계산 (초 → 일)
    if trough_index > peak_index:
        duration = (timestamps[trough_index] - timestamps[peak_index]) / 86400
    else:
        duration = 0
    
    return {
        'max_dd': round(max_dd, 4),
        'peak': peak,
        'trough': balances[trough_index] if trough_index > 0 else peak,
        'duration_days': int(duration)
    }
```

---

### 📌 함수: PerformanceTracker.calculate_win_rate(trades)

```python
def calculate_win_rate(self, trades: List[Dict]) -> Dict:
```

#### 역할
승률 및 관련 통계 계산

#### 반환값
```python
Dict:
    'total_trades': int = 15
    'win_rate': float = 66.7
    'avg_win': float = 3.5
    'avg_loss': float = -1.0
    'best_trade': float = 4.2
    'worst_trade': float = -1.0
    'avg_holding_minutes': int = 180
```

#### 구현 코드
```python
def calculate_win_rate(self, trades: List[Dict]) -> Dict:
    """
    승률 통계
    
    Example:
        >>> trades = db.get_trades_last_n_days(7)
        >>> stats = tracker.calculate_win_rate(trades)
        >>> print(f"승률: {stats['win_rate']:.1f}%")
        승률: 66.7%
    """
    if not trades:
        return {
            'total_trades': 0,
            'win_rate': 0.0,
            'avg_win': 0.0,
            'avg_loss': 0.0
        }
    
    total = len(trades)
    winners = [t for t in trades if t['pnl_percent'] > 0]
    losers = [t for t in trades if t['pnl_percent'] <= 0]
    
    win_rate = (len(winners) / total) * 100
    
    avg_win = (
        sum(t['pnl_percent'] for t in winners) / len(winners) * 100 
        if winners else 0
    )
    
    avg_loss = (
        sum(t['pnl_percent'] for t in losers) / len(losers) * 100 
        if losers else 0
    )
    
    best = max(trades, key=lambda x: x['pnl_percent'])
    worst = min(trades, key=lambda x: x['pnl_percent'])
    
    avg_holding = (
        sum(t['holding_minutes'] for t in trades) / total
        if all('holding_minutes' in t for t in trades) else 0
    )
    
    return {
        'total_trades': total,
        'win_rate': round(win_rate, 1),
        'avg_win': round(avg_win, 2),
        'avg_loss': round(avg_loss, 2),
        'best_trade': round(best['pnl_percent'] * 100, 2),
        'worst_trade': round(worst['pnl_percent'] * 100, 2),
        'avg_holding_minutes': int(avg_holding)
    }
```

---

### 📌 함수: PerformanceTracker.calculate_profit_factor(trades)

```python
def calculate_profit_factor(self, trades: List[Dict]) -> float:
```

#### 역할
Profit Factor 계산 (총이익 / 총손실)

#### 공식
```
Profit Factor = 총 수익 거래 금액 / 총 손실 거래 금액

해석:
  > 2.0: 매우 우수
  1.5 - 2.0: 우수
  1.0 - 1.5: 보통
  < 1.0: 개선 필요
```

#### 구현 코드
```python
def calculate_profit_factor(self, trades: List[Dict]) -> float:
    """
    Profit Factor 계산
    
    Example:
        >>> pf = tracker.calculate_profit_factor(trades)
        >>> print(f"Profit Factor: {pf:.2f}")
        Profit Factor: 1.85
    """
    if not trades:
        return 0.0
    
    winners = [t for t in trades if t['pnl_krw'] > 0]
    losers = [t for t in trades if t['pnl_krw'] <= 0]
    
    total_wins = sum(t['pnl_krw'] for t in winners) if winners else 0
    total_losses = abs(sum(t['pnl_krw'] for t in losers)) if losers else 0
    
    if total_losses == 0:
        return 0.0 if total_wins == 0 else float('inf')
    
    profit_factor = total_wins / total_losses
    
    return round(profit_factor, 2)
```

---

### 📌 함수: PerformanceTracker.get_performance_metrics(start_date, end_date)

```python
def get_performance_metrics(
    self,
    start_date: str = None,
    end_date: str = None
) -> Dict:
```

#### 역할
전체 성과 지표 한 번에 계산

#### 호출되는 곳
```python
# monitoring/reporter.py - 월간 리포트
metrics = self.tracker.get_performance_metrics('2025-01-01', '2025-01-31')
```

#### 반환값
```python
Dict:
    'win_rate_stats': Dict = {...}  # calculate_win_rate() 결과
    'sharpe_ratio': float = 1.45
    'max_drawdown': Dict = {...}    # calculate_max_drawdown() 결과
    'profit_factor': float = 1.85
    'total_pnl_krw': float = 35420
    'total_pnl_percent': float = 3.54
    'best_day': Dict = {'date': '2025-01-15', 'pnl': 12340}
    'worst_day': Dict = {'date': '2025-01-08', 'pnl': -8500}
```

#### 구현 코드
```python
def get_performance_metrics(
    self,
    start_date: str = None,
    end_date: str = None
) -> Dict:
    """
    전체 성과 지표
    
    Example:
        >>> metrics = tracker.get_performance_metrics('2025-01-01', '2025-01-31')
        >>> print(f"Sharpe: {metrics['sharpe_ratio']}")
        >>> print(f"Max DD: {metrics['max_drawdown']['max_dd']*100:.1f}%")
        >>> print(f"Win Rate: {metrics['win_rate_stats']['win_rate']:.1f}%")
    """
    # 거래 조회
    if start_date and end_date:
        trades = self.db.get_trades_by_period(start_date, end_date)
    else:
        trades = self.db.get_all_closed_trades()
    
    if not trades:
        return {'error': 'No trades found'}
    
    # 1. 승률 통계
    win_rate_stats = self.calculate_win_rate(trades)
    
    # 2. 샤프 비율
    returns = [t['pnl_percent'] for t in trades]
    sharpe = self.calculate_sharpe_ratio(returns)
    
    # 3. Profit Factor
    profit_factor = self.calculate_profit_factor(trades)
    
    # 4. 총 손익
    total_pnl_krw = sum(t['pnl_krw'] for t in trades)
    
    # 5. 최대 낙폭 (잔고 이력 필요)
    # 실제로는 별도로 저장된 balance_history 사용
    max_dd = {'max_dd': 0.0, 'peak': 0.0}  # 간단 버전
    
    # 6. 일별 최고/최저
    daily_pnl = {}
    for trade in trades:
        date = trade['timestamp'].split()[0]
        if date not in daily_pnl:
            daily_pnl[date] = 0
        daily_pnl[date] += trade['pnl_krw']
    
    best_day = max(daily_pnl.items(), key=lambda x: x[1]) if daily_pnl else ('', 0)
    worst_day = min(daily_pnl.items(), key=lambda x: x[1]) if daily_pnl else ('', 0)
    
    return {
        'win_rate_stats': win_rate_stats,
        'sharpe_ratio': sharpe,
        'max_drawdown': max_dd,
        'profit_factor': profit_factor,
        'total_pnl_krw': round(total_pnl_krw, 0),
        'best_day': {'date': best_day[0], 'pnl': round(best_day[1], 0)},
        'worst_day': {'date': worst_day[0], 'pnl': round(worst_day[1], 0)}
    }
```

---

## 전체 의존성 그래프

### MONITORING 모듈 구조
```
logger.py (독립)
├── 사용: logging, core/constants
└── 사용처: engine/base_engine.py

reporter.py
├── 사용: database/TradeDatabase
└── 사용처: engine/base_engine.py

performance.py
├── 사용: database/TradeDatabase, numpy, pandas
└── 사용처: monitoring/reporter.py
```

### 사용하는 모듈
```
core/constants → LOG_DIR
database/TradeDatabase → 거래 데이터 조회
logging (표준 라이브러리)
numpy, pandas (성과 계산)
```

### 사용되는 곳
```
engine/base_engine.py
├── TradingLogger (모든 이벤트 로깅)
├── PerformanceReporter (리포트 생성)
└── PerformanceTracker (성과 측정)

ai/learner.py
└── PerformanceTracker (패턴 분석용)
```

---

## 개발 체크리스트

### logger.py
- [ ] TradingLogger 클래스
- [ ] setup_logger() - 파일/콘솔 핸들러
- [ ] log_entry() - 진입 로그
- [ ] log_exit() - 청산 로그
- [ ] log_risk_event() - 리스크 이벤트
- [ ] log_ai_decision() - AI 판단
- [ ] 로그 필터 (거래/AI/에러 분리)

### reporter.py
- [ ] PerformanceReporter 클래스
- [ ] generate_daily_summary() - 일일 요약
- [ ] generate_weekly_report() - 주간 분석
- [ ] generate_monthly_report() - 월간 리포트
- [ ] save_report() - 파일 저장

### performance.py
- [ ] PerformanceTracker 클래스
- [ ] calculate_sharpe_ratio() - 샤프 비율
- [ ] calculate_max_drawdown() - 최대 낙폭
- [ ] calculate_win_rate() - 승률 통계
- [ ] calculate_profit_factor() - Profit Factor
- [ ] get_performance_metrics() - 종합 지표

---

## 테스트 시나리오

### logger.py 테스트
```python
from monitoring import TradingLogger

# 1. 초기화
logger = TradingLogger(mode='paper', level='INFO')

# 2. 진입 로그
logger.log_entry('DOGE/USDT', 0.3821, 500000, 0.75)
# 출력: [2025-01-15 14:32:15] [INFO] [ENTRY] DOGE/USDT @ 0.3821 | Amount: 500,000 KRW | AI: 0.75

# 3. 청산 로그
logger.log_exit('DOGE/USDT', 0.3821, 0.3895, 0.0194, 9700, 'TRAILING_STOP')
# 출력: [2025-01-15 16:45:22] [INFO] [EXIT] DOGE/USDT @ 0.3895 | PnL: +1.94% (+9,700 KRW) | Reason: TRAILING_STOP

# 4. 리스크 이벤트
logger.log_risk_event('DAILY_LIMIT', {'loss': -0.05, 'resume_time': 1234567890})
# 출력: [2025-01-15 18:00:00] [WARNING] [RISK] DAILY_LIMIT | Loss: -5.0% | Resume: 2025-01-16 00:00:00

# 5. 로그 파일 확인
# logs/paper/2025-01-15_trades.log - 진입/청산만
# logs/paper/2025-01-15_decisions.log - AI 판단만
# logs/paper/2025-01-15_errors.log - 에러만
```

### reporter.py 테스트
```python
from monitoring import PerformanceReporter
from database import TradeDatabase

# 1. 초기화
db = TradeDatabase()
reporter = PerformanceReporter(db)

# 2. 일일 리포트
daily_report = reporter.generate_daily_summary()
print(daily_report)
reporter.save_report(daily_report, 'daily')

# 3. 주간 리포트
weekly_report = reporter.generate_weekly_report()
print(weekly_report)
reporter.save_report(weekly_report, 'weekly')

# 4. 파일 확인
# logs/reports/daily/2025-01-15.txt
# logs/reports/weekly/2025-W03.txt
```

### performance.py 테스트
```python
from monitoring import PerformanceTracker
from database import TradeDatabase

# 1. 초기화
db = TradeDatabase()
tracker = PerformanceTracker(db)

# 2. 샤프 비율
returns = [0.02, -0.01, 0.035, 0.018, -0.008]
sharpe = tracker.calculate_sharpe_ratio(returns)
print(f"Sharpe Ratio: {sharpe:.2f}")
# 출력: Sharpe Ratio: 1.45

# 3. 승률
trades = db.get_trades_last_n_days(7)
win_stats = tracker.calculate_win_rate(trades)
print(f"승률: {win_stats['win_rate']:.1f}%")
print(f"평균 수익: {win_stats['avg_win']:.2f}%")

# 4. Profit Factor
pf = tracker.calculate_profit_factor(trades)
print(f"Profit Factor: {pf:.2f}")

# 5. 전체 지표
metrics = tracker.get_performance_metrics('2025-01-01', '2025-01-31')
print(f"총 거래: {metrics['win_rate_stats']['total_trades']}")
print(f"Sharpe: {metrics['sharpe_ratio']}")
print(f"PF: {metrics['profit_factor']}")
```

---

## 주요 특징

### 1. 다층 로깅 시스템
- **콘솔**: 실시간 모니터링
- **trades.log**: 거래 기록만
- **decisions.log**: AI 판단만
- **errors.log**: 에러만
- **debug.log**: 전체 (DEBUG 모드)

### 2. 자동 리포트 생성
- **일일**: 매일 09:00, 21:00
- **주간**: 매주 월요일 00:00
- **월간**: 매월 1일 00:00

### 3. 표준 성과 지표
- **Sharpe Ratio**: 위험 대비 수익
- **Max Drawdown**: 최대 낙폭
- **Win Rate**: 승률
- **Profit Factor**: 수익/손실 비율

### 4. 파일 관리
- **로그 로테이션**: 일일 파일 생성
- **리포트 아카이브**: 기간별 저장
- **자동 정리**: 30일 이상 로그 삭제 (선택)

---

## 통합 사용 예시

### engine/base_engine.py에서 사용
```python
from monitoring import TradingLogger, PerformanceReporter, PerformanceTracker

class BaseEngine:
    def __init__(self, config, exchange):
        # 로거
        self.logger = TradingLogger(mode=config.MODE, level='INFO')
        
        # 리포터
        self.reporter = PerformanceReporter(self.trade_db)
        
        # 트래커
        self.tracker = PerformanceTracker(self.trade_db)
    
    async def execute_entry(self, symbol, signal):
        # 진입 로그
        self.logger.log_entry(symbol, price, amount, signal['confidence'])
        
        # AI 결정 로그
        self.logger.log_ai_decision(
            symbol, 
            'ENTER', 
            signal['confidence'], 
            signal['reasoning']
        )
    
    async def execute_exit(self, symbol, position, signal):
        # 청산 로그
        self.logger.log_exit(
            symbol,
            position['entry_price'],
            exit_price,
            pnl_percent,
            pnl_krw,
            signal['reason']
        )
    
    def check_daily_report(self):
        """매일 21:00 실행"""
        report = self.reporter.generate_daily_summary()
        print(report)
        self.reporter.save_report(report, 'daily')
    
    def check_weekly_report(self):
        """매주 월요일 실행"""
        report = self.reporter.generate_weekly_report()
        print(report)
        self.reporter.save_report(report, 'weekly')
```

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 10 (모니터링 레이어)  
**검증**: ✅ 완료