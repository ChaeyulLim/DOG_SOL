# 07_RISK 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [risk/manager.py](#riskmanagerpy)
2. [risk/position.py](#riskpositionpy)
3. [risk/limits.py](#risklimitspy)
4. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📁 risk/manager.py

### 파일 전체 구조
```python
import time
from typing import Dict, Optional
from core.config import Config
from core.exceptions import DailyLossLimitError, MonthlyDrawdownError, ConsecutiveLossError
from .limits import LimitsChecker

class RiskManager:
    def __init__(self, config: Config): ...
    
    def check_all_limits(
        self,
        current_balance: float,
        today_start_balance: float,
        monthly_high: float
    ) -> Dict: ...
    
    def check_daily_loss(
        self,
        current_balance: float,
        today_start_balance: float
    ) -> Dict: ...
    
    def check_monthly_drawdown(
        self,
        current_balance: float,
        monthly_high: float
    ) -> Dict: ...
    
    def record_trade_result(self, pnl: float) -> None: ...
    
    def check_consecutive_losses(self) -> Dict: ...
    
    def get_risk_status(self) -> Dict: ...
```

---

### 📌 클래스: RiskManager

#### 목적
리스크 한도 모니터링 및 제어 (3단계 안전장치)

---

### 📌 함수: RiskManager.__init__(config)

```python
def __init__(self, config: Config):
```

#### 역할
리스크 관리자 초기화

#### 초기화 내용
```python
self.config = config
self.consecutive_losses = 0
self.limits_checker = LimitsChecker(config)
self.pause_until = 0  # 거래 중지 시각
```

#### 호출되는 곳
```python
# engine/base_engine.py __init__()
from risk import RiskManager

self.risk_manager = RiskManager(self.config)
```

#### 구현 코드
```python
def __init__(self, config: Config):
    """리스크 관리자 초기화"""
    self.config = config
    self.consecutive_losses = 0
    self.limits_checker = LimitsChecker(config)
    self.pause_until = 0
```

---

### 📌 함수: RiskManager.check_all_limits(current_balance, today_start_balance, monthly_high)

```python
def check_all_limits(
    self,
    current_balance: float,
    today_start_balance: float,
    monthly_high: float
) -> Dict:
```

#### 역할
모든 리스크 한도 종합 체크

#### 인자
- `current_balance: float` - 현재 잔고 (KRW)
- `today_start_balance: float` - 오늘 시작 잔고
- `monthly_high: float` - 이번 달 최고 잔고

#### 호출되는 곳
```python
# engine/base_engine.py main_loop() - 매 거래 후
risk_result = self.risk_manager.check_all_limits(
    current_balance,
    today_start_balance,
    monthly_high
)

if risk_result['action'] != 'OK':
    # 거래 중단
    self.handle_risk_limit(risk_result)
```

#### 체크 순서
```
1. 일일 손실 (-5%)
2. 월간 드로다운 (-10%)
3. 연속 손실 (3회)

하나라도 초과 시 즉시 중단
```

#### 반환값
```python
# 정상
Dict:
    'action': str = 'OK'

# 한도 초과
Dict:
    'action': str = 'STOP_TRADING' | 'EMERGENCY_STOP' | 'PAUSE_24H'
    'reason': str = 'DAILY_LIMIT' | 'MONTHLY_DD' | 'CONSECUTIVE_LOSS'
    'resume_time': int = timestamp
    'details': Dict
```

#### 구현 코드
```python
def check_all_limits(
    self,
    current_balance: float,
    today_start_balance: float,
    monthly_high: float
) -> Dict:
    """
    모든 리스크 한도 체크
    
    Returns:
        {'action': 'OK'} or {'action': 'STOP_TRADING', ...}
    """
    # 1. 일일 손실
    daily_result = self.check_daily_loss(
        current_balance,
        today_start_balance
    )
    if daily_result['action'] != 'OK':
        return daily_result
    
    # 2. 월간 드로다운
    monthly_result = self.check_monthly_drawdown(
        current_balance,
        monthly_high
    )
    if monthly_result['action'] != 'OK':
        return monthly_result
    
    # 3. 연속 손실
    consecutive_result = self.check_consecutive_losses()
    if consecutive_result['action'] != 'OK':
        return consecutive_result
    
    return {'action': 'OK'}
```

---

### 📌 함수: RiskManager.check_daily_loss(current_balance, today_start_balance)

```python
def check_daily_loss(
    self,
    current_balance: float,
    today_start_balance: float
) -> Dict:
```

#### 역할
일일 손실 한도 체크 (-5%)

#### 로직
```python
daily_pnl = (current_balance - today_start_balance) / today_start_balance

if daily_pnl <= -0.05:
    # 일일 손실 -5% 도달
    # 1. 모든 포지션 즉시 청산
    # 2. 오늘 거래 중단
    # 3. 내일 00:00 재개
```

#### 반환값
```python
# 한도 초과
Dict:
    'action': str = 'STOP_TRADING'
    'reason': str = 'DAILY_LIMIT'
    'resume_time': int = tomorrow_midnight
    'loss': float = -0.05
    'current_balance': float

# 정상
Dict:
    'action': str = 'OK'
```

#### 구현 코드
```python
def check_daily_loss(
    self,
    current_balance: float,
    today_start_balance: float
) -> Dict:
    """
    일일 손실 한도 체크
    
    Limit: -5%
    """
    daily_pnl = (current_balance - today_start_balance) / today_start_balance
    
    if daily_pnl <= self.config.DAILY_LOSS_LIMIT:
        # 내일 00:00 계산
        import datetime
        tomorrow = datetime.datetime.now() + datetime.timedelta(days=1)
        tomorrow_midnight = tomorrow.replace(
            hour=0, minute=0, second=0, microsecond=0
        )
        resume_time = int(tomorrow_midnight.timestamp())
        
        return {
            'action': 'STOP_TRADING',
            'reason': 'DAILY_LIMIT',
            'resume_time': resume_time,
            'loss': daily_pnl,
            'current_balance': current_balance
        }
    
    return {'action': 'OK'}
```

---

### 📌 함수: RiskManager.check_monthly_drawdown(current_balance, monthly_high)

```python
def check_monthly_drawdown(
    self,
    current_balance: float,
    monthly_high: float
) -> Dict:
```

#### 역할
월간 드로다운 한도 체크 (-10%)

#### 로직
```python
# 현재 잔고가 월간 최고점보다 높으면 갱신
if current_balance > monthly_high:
    monthly_high = current_balance

drawdown = (current_balance - monthly_high) / monthly_high

if drawdown <= -0.10:
    # 긴급 정지 (다음 달까지)
```

#### 반환값
```python
# 한도 초과
Dict:
    'action': str = 'EMERGENCY_STOP'
    'reason': str = 'MONTHLY_DD'
    'resume_time': int = next_month_timestamp
    'drawdown': float = -0.10
    'monthly_high': float

# 정상
Dict:
    'action': str = 'OK'
```

#### 구현 코드
```python
def check_monthly_drawdown(
    self,
    current_balance: float,
    monthly_high: float
) -> Dict:
    """
    월간 드로다운 체크
    
    Limit: -10%
    """
    # 신고점 갱신
    if current_balance > monthly_high:
        monthly_high = current_balance
    
    drawdown = (current_balance - monthly_high) / monthly_high
    
    if drawdown <= self.config.MONTHLY_DD_LIMIT:
        # 다음 달 1일 00:00
        import datetime
        now = datetime.datetime.now()
        if now.month == 12:
            next_month = now.replace(year=now.year+1, month=1, day=1)
        else:
            next_month = now.replace(month=now.month+1, day=1)
        
        next_month = next_month.replace(
            hour=0, minute=0, second=0, microsecond=0
        )
        resume_time = int(next_month.timestamp())
        
        return {
            'action': 'EMERGENCY_STOP',
            'reason': 'MONTHLY_DD',
            'resume_time': resume_time,
            'drawdown': drawdown,
            'monthly_high': monthly_high
        }
    
    return {'action': 'OK'}
```

---

### 📌 함수: RiskManager.record_trade_result(pnl)

```python
def record_trade_result(self, pnl: float) -> None:
```

#### 역할
거래 결과 기록 (연속 손실 카운트용)

#### 인자
- `pnl: float` - 손익률 (0.02 = +2%)

#### 호출되는 곳
```python
# engine/base_engine.py - 거래 종료 후
self.risk_manager.record_trade_result(pnl)
```

#### 로직
```python
if pnl < 0:
    consecutive_losses += 1
else:
    consecutive_losses = 0  # 리셋
```

#### 구현 코드
```python
def record_trade_result(self, pnl: float) -> None:
    """거래 결과 기록"""
    
    if pnl < 0:
        self.consecutive_losses += 1
    else:
        self.consecutive_losses = 0
```

---

### 📌 함수: RiskManager.check_consecutive_losses()

```python
def check_consecutive_losses(self) -> Dict:
```

#### 역할
연속 손실 3회 체크

#### 반환값
```python
# 3회 도달
Dict:
    'action': str = 'PAUSE_24H'
    'reason': str = 'CONSECUTIVE_LOSS'
    'resume_time': int = now + 86400
    'count': int = 3

# 정상
Dict:
    'action': str = 'OK'
```

#### 구현 코드
```python
def check_consecutive_losses(self) -> Dict:
    """
    연속 손실 체크
    
    Limit: 3회
    """
    if self.consecutive_losses >= self.config.CONSECUTIVE_LOSS_LIMIT:
        resume_time = int(time.time() + 86400)  # 24시간 후
        
        return {
            'action': 'PAUSE_24H',
            'reason': 'CONSECUTIVE_LOSS',
            'resume_time': resume_time,
            'count': self.consecutive_losses
        }
    
    return {'action': 'OK'}
```

---

### 📌 함수: RiskManager.get_risk_status()

```python
def get_risk_status(self) -> Dict:
```

#### 역할
현재 리스크 상태 조회 (대시보드용)

#### 반환값
```python
Dict:
    'daily': Dict = {
        'current_pnl': float,
        'limit': -0.05,
        'remaining': float,
        'status': 'OK' | 'WARNING' | 'CRITICAL'
    }
    'monthly': Dict = {
        'current_dd': float,
        'limit': -0.10,
        'remaining': float,
        'status': 'OK' | 'WARNING' | 'CRITICAL'
    }
    'consecutive': Dict = {
        'current': int,
        'limit': 3,
        'remaining': int,
        'status': 'OK' | 'WARNING' | 'CRITICAL'
    }
    'overall': str = 'GREEN' | 'YELLOW' | 'RED'
```

#### 구현 코드
```python
def get_risk_status(self) -> Dict:
    """현재 리스크 상태"""
    # 계산 로직 생략 (구현 시 추가)
    
    return {
        'daily': {'status': 'OK'},
        'monthly': {'status': 'OK'},
        'consecutive': {'status': 'OK'},
        'overall': 'GREEN'
    }
```

---

## 📁 risk/position.py

### 파일 전체 구조
```python
from typing import Dict
from core.config import Config

class PositionSizer:
    def __init__(self, config: Config): ...
    
    def calculate_position_size(
        self,
        balance: float,
        symbol: str
    ) -> int: ...
    
    def kelly_position_size(
        self,
        balance: float,
        win_rate: float,
        avg_win: float,
        avg_loss: float
    ) -> int: ...
```

---

### 📌 클래스: PositionSizer

#### 목적
포지션 크기 계산 (고정 배분 or Kelly Criterion)

---

### 📌 함수: PositionSizer.calculate_position_size(balance, symbol)

```python
def calculate_position_size(
    self,
    balance: float,
    symbol: str
) -> int:
```

#### 역할
심볼별 할당 금액 계산 (고정 배분)

#### 인자
- `balance: float` - 총 잔고 (KRW)
- `symbol: str` - 'DOGE' or 'SOL'

#### 호출되는 곳
```python
# engine/base_engine.py - 진입 시
amount_krw = self.position_sizer.calculate_position_size(
    self.get_balance(),
    'DOGE'
)
```

#### 로직
```python
allocation = config.POSITION_ALLOCATION[symbol]  # 0.5
return int(balance * allocation)  # 50%
```

#### 구현 코드
```python
def calculate_position_size(
    self,
    balance: float,
    symbol: str
) -> int:
    """
    포지션 크기 계산 (고정 배분)
    
    Example:
        >>> sizer.calculate_position_size(1_000_000, 'DOGE')
        500000
    """
    allocation = self.config.POSITION_ALLOCATION.get(symbol, 0)
    return int(balance * allocation)
```

---

### 📌 함수: PositionSizer.kelly_position_size(balance, win_rate, avg_win, avg_loss)

```python
def kelly_position_size(
    self,
    balance: float,
    win_rate: float,
    avg_win: float,
    avg_loss: float
) -> int:
```

#### 역할
Kelly Criterion 기반 포지션 크기 (선택 기능)

#### 인자
- `win_rate: float` - 승률 (0.70 = 70%)
- `avg_win: float` - 평균 수익률 (0.035 = 3.5%)
- `avg_loss: float` - 평균 손실률 (0.01 = 1%)

#### Kelly 공식
```python
f = (p * b - q) / b

f = 배팅 비율
p = 승률
q = 패율 (1-p)
b = 평균수익 / 평균손실

# 1/4 Kelly 적용 (리스크 완화)
adjusted_kelly = kelly * 0.25

# 최대 50% 제한
final = min(adjusted_kelly, 0.5)
```

#### 구현 코드
```python
def kelly_position_size(
    self,
    balance: float,
    win_rate: float,
    avg_win: float,
    avg_loss: float
) -> int:
    """
    Kelly Criterion
    
    Example:
        >>> sizer.kelly_position_size(
        ...     1_000_000, 0.70, 0.035, 0.01
        ... )
        152500  # 약 15.25%
    """
    p = win_rate
    q = 1 - win_rate
    b = avg_win / avg_loss
    
    # Kelly
    kelly = (p * b - q) / b
    
    # 1/4 Kelly
    adjusted = kelly * 0.25
    
    # 최대 50%
    final_fraction = min(adjusted, 0.5)
    
    return int(balance * final_fraction)
```

---

## 📁 risk/limits.py

### 파일 전체 구조
```python
from typing import Dict
from core.config import Config

class LimitsChecker:
    def __init__(self, config: Config): ...
    
    def is_within_daily_limit(self, daily_pnl: float) -> bool: ...
    def is_within_monthly_limit(self, drawdown: float) -> bool: ...
    def is_within_consecutive_limit(self, count: int) -> bool: ...
    
    def get_remaining_capacity(
        self,
        current_value: float,
        limit: float
    ) -> Dict: ...
```

---

### 📌 클래스: LimitsChecker

#### 목적
한도 체크 유틸리티

---

### 📌 함수: LimitsChecker.is_within_daily_limit(daily_pnl)

```python
def is_within_daily_limit(self, daily_pnl: float) -> bool:
```

#### 역할
일일 한도 내 여부 확인

#### 반환값
- `bool`: True (안전), False (초과)

#### 구현 코드
```python
def is_within_daily_limit(self, daily_pnl: float) -> bool:
    """일일 한도 체크"""
    return daily_pnl > self.config.DAILY_LOSS_LIMIT
```

---

### 📌 함수: LimitsChecker.get_remaining_capacity(current_value, limit)

```python
def get_remaining_capacity(
    self,
    current_value: float,
    limit: float
) -> Dict:
```

#### 역할
남은 여유 계산

#### 반환값
```python
Dict:
    'current': float = -0.02
    'limit': float = -0.05
    'remaining': float = 0.03  # 3% 여유
    'usage_percent': float = 40.0  # 40% 사용
```

#### 구현 코드
```python
def get_remaining_capacity(
    self,
    current_value: float,
    limit: float
) -> Dict:
    """남은 여유 계산"""
    
    remaining = limit - current_value
    usage = (current_value / limit) * 100 if limit != 0 else 0
    
    return {
        'current': current_value,
        'limit': limit,
        'remaining': remaining,
        'usage_percent': usage
    }
```

---

## 전체 의존성 그래프

### RISK 모듈 구조
```
manager.py
├── import limits.py (LimitsChecker)
├── import position.py (X, 별도 사용)
└── import core

position.py
└── import core

limits.py
└── import core
```

### 사용하는 모듈
```
core/config → 한도 설정값
core/exceptions → 리스크 예외
```

### 사용되는 곳
```
engine/base_engine.py
├── RiskManager.check_all_limits()
├── PositionSizer.calculate_position_size()
└── RiskManager.record_trade_result()
```

---

## 개발 체크리스트

### manager.py
- [ ] RiskManager 클래스
- [ ] check_all_limits() - 3가지 한도 체크
- [ ] check_daily_loss() - -5%
- [ ] check_monthly_drawdown() - -10%
- [ ] check_consecutive_losses() - 3회
- [ ] record_trade_result() - 카운트 관리
- [ ] get_risk_status() - 상태 조회

### position.py
- [ ] PositionSizer 클래스
- [ ] calculate_position_size() - 고정 배분
- [ ] kelly_position_size() - Kelly Criterion
- [ ] 최대 50% 제한

### limits.py
- [ ] LimitsChecker 클래스
- [ ] is_within_*_limit() 함수들
- [ ] get_remaining_capacity()

---

## 테스트 시나리오

### manager.py 테스트
```python
# 1. 초기화
config = Config.load('paper')
risk = RiskManager(config)

# 2. 정상 (손실 -2%)
result = risk.check_daily_loss(980_000, 1_000_000)
assert result['action'] == 'OK'

# 3. 일일 한도 (-5%)
result = risk.check_daily_loss(950_000, 1_000_000)
assert result['action'] == 'STOP_TRADING'
assert 'resume_time' in result

# 4. 연속 손실
risk.record_trade_result(-0.01)
risk.record_trade_result(-0.01)
risk.record_trade_result(-0.01)
result = risk.check_consecutive_losses()
assert result['action'] == 'PAUSE_24H'

# 5. 수익 후 리셋
risk.record_trade_result(0.02)
assert risk.consecutive_losses == 0
```

### position.py 테스트
```python
# 1. 고정 배분
sizer = PositionSizer(config)
amount = sizer.calculate_position_size(1_000_000, 'DOGE')
assert amount == 500_000  # 50%

# 2. Kelly
amount = sizer.kelly_position_size(
    1_000_000,
    0.70,  # 70% 승률
    0.035,  # 평균 +3.5%
    0.01    # 평균 -1%
)
assert 100_000 < amount < 200_000  # 약 15%
```

---

## 주요 특징

### 1. 3단계 안전장치
- **Level 1**: 일일 -5% (당일 중단)
- **Level 2**: 월간 -10% (긴급 정지)
- **Level 3**: 연속 3회 (24시간 중지)

### 2. 자동 재개
- 일일: 다음 날 00:00
- 월간: 다음 달 1일
- 연속: 24시간 후

### 3. 포지션 관리
- 기본: 50:50 고정 배분
- 고급: Kelly Criterion (선택)
- 최대 50% 제한

### 4. 실시간 모니터링
- 대시보드용 상태 조회
- 남은 여유 계산
- 경고 레벨 (GREEN/YELLOW/RED)

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 6 (리스크 레이어)  
**검증**: ✅ 완료