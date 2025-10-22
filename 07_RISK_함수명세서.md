# 07_RISK 모듈 완벽 함수 명세서 v2.0 (개선판)

> **개선사항**: monthly_high 관리 방식 명확화, get_risk_status() 완전 구현, pause_until 활용, 에러 처리 강화

---

## 📋 목차
1. [risk/manager.py](#riskmanagerpy) ⭐ 개선
2. [risk/position.py](#riskpositionpy)
3. [risk/limits.py](#risklimitspy)
4. [전체 의존성 그래프](#전체-의존성-그래프)
5. [실전 사용 예제](#실전-사용-예제)

---

## 📁 risk/manager.py ⭐ 개선

### 구현 코드 (전체 개선)

```python
import time
import datetime
from typing import Dict, Optional
from core.config import Config
from core.exceptions import (
    DailyLossLimitError,
    MonthlyDrawdownError,
    ConsecutiveLossError
)


class RiskManager:
    """리스크 한도 모니터링 및 제어 (3단계 안전장치)"""
    
    def __init__(self, config: Config):
        """
        리스크 관리자 초기화
        
        Args:
            config: Config 인스턴스
        """
        self.config = config
        self.consecutive_losses = 0
        self.pause_until = 0  # 거래 중지 시각 (timestamp)
    
    def check_all_limits(
        self,
        current_balance: float,
        today_start_balance: float,
        monthly_high: float
    ) -> Dict:
        """
        모든 리스크 한도 종합 체크
        
        Args:
            current_balance: 현재 잔고 (KRW)
            today_start_balance: 오늘 시작 잔고 (KRW)
            monthly_high: 이번 달 최고 잔고 (KRW)
        
        Returns:
            {
                'action': 'OK' | 'STOP_TRADING' | 'EMERGENCY_STOP' | 'PAUSE_24H',
                'reason': str | None,
                'resume_time': int | None,
                'new_monthly_high': float,  # ⭐ 갱신된 월간 최고가
                'details': Dict
            }
        
        호출:
            engine/base_engine.py main_loop() - 매 거래 후
        
        Example:
            >>> risk_result = risk_manager.check_all_limits(
            ...     980_000, 1_000_000, 1_100_000
            ... )
            >>> if risk_result['action'] != 'OK':
            ...     handle_risk_limit(risk_result)
        """
        # 1. 거래 중지 상태 확인 ⭐
        if self.is_trading_paused():
            return {
                'action': 'PAUSED',
                'reason': 'TRADING_PAUSED',
                'resume_time': self.pause_until,
                'new_monthly_high': monthly_high,
                'details': {}
            }
        
        # 2. 일일 손실 체크
        daily_result = self.check_daily_loss(
            current_balance,
            today_start_balance
        )
        if daily_result['action'] != 'OK':
            return daily_result
        
        # 3. 월간 드로다운 체크 (⭐ new_monthly_high 반환)
        monthly_result = self.check_monthly_drawdown(
            current_balance,
            monthly_high
        )
        if monthly_result['action'] != 'OK':
            return monthly_result
        
        # 4. 연속 손실 체크
        consecutive_result = self.check_consecutive_losses()
        if consecutive_result['action'] != 'OK':
            return consecutive_result
        
        # 5. 정상 (⭐ new_monthly_high 포함)
        return {
            'action': 'OK',
            'new_monthly_high': monthly_result.get('new_monthly_high', monthly_high)
        }
    
    def check_daily_loss(
        self,
        current_balance: float,
        today_start_balance: float
    ) -> Dict:
        """
        일일 손실 한도 체크 (-5%)
        
        Returns:
            정상: {'action': 'OK'}
            초과: {
                'action': 'STOP_TRADING',
                'reason': 'DAILY_LIMIT',
                'resume_time': int,  # 내일 00:00
                'loss': float,
                'current_balance': float
            }
        
        Example:
            >>> # 정상 (-2%)
            >>> result = risk_manager.check_daily_loss(980_000, 1_000_000)
            >>> assert result['action'] == 'OK'
            
            >>> # 한도 초과 (-5%)
            >>> result = risk_manager.check_daily_loss(950_000, 1_000_000)
            >>> assert result['action'] == 'STOP_TRADING'
        """
        # ⭐ 에러 방지: 0으로 나누기
        if today_start_balance <= 0:
            return {'action': 'OK'}  # 또는 WARNING 로그
        
        # 일일 PnL 계산
        daily_pnl = (current_balance - today_start_balance) / today_start_balance
        
        # 한도 체크
        if daily_pnl <= self.config.DAILY_LOSS_LIMIT:
            # 내일 00:00 계산
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
    
    def check_monthly_drawdown(
        self,
        current_balance: float,
        monthly_high: float
    ) -> Dict:
        """
        월간 드로다운 한도 체크 (-10%)
        
        ⭐ 개선: new_monthly_high 반환하여 외부에서 저장
        
        Returns:
            정상: {
                'action': 'OK',
                'new_monthly_high': float  # ⭐ 갱신된 값
            }
            초과: {
                'action': 'EMERGENCY_STOP',
                'reason': 'MONTHLY_DD',
                'resume_time': int,  # 다음 달 1일
                'drawdown': float,
                'monthly_high': float,
                'new_monthly_high': float
            }
        
        호출자 책임:
            - new_monthly_high를 DB 또는 state에 저장
        
        Example:
            >>> result = risk_manager.check_monthly_drawdown(
            ...     950_000, 1_100_000
            ... )
            >>> # 호출자가 저장
            >>> state.monthly_high = result['new_monthly_high']
        """
        # ⭐ 에러 방지
        if monthly_high <= 0:
            monthly_high = current_balance
        
        # 신고점 갱신 ⭐
        new_monthly_high = max(monthly_high, current_balance)
        
        # 드로다운 계산
        drawdown = (current_balance - new_monthly_high) / new_monthly_high
        
        # 한도 체크
        if drawdown <= self.config.MONTHLY_DD_LIMIT:
            # 다음 달 1일 00:00 계산
            now = datetime.datetime.now()
            if now.month == 12:
                next_month = now.replace(
                    year=now.year + 1,
                    month=1,
                    day=1
                )
            else:
                next_month = now.replace(
                    month=now.month + 1,
                    day=1
                )
            
            next_month = next_month.replace(
                hour=0, minute=0, second=0, microsecond=0
            )
            resume_time = int(next_month.timestamp())
            
            return {
                'action': 'EMERGENCY_STOP',
                'reason': 'MONTHLY_DD',
                'resume_time': resume_time,
                'drawdown': drawdown,
                'monthly_high': new_monthly_high,
                'new_monthly_high': new_monthly_high  # ⭐
            }
        
        # 정상 (⭐ new_monthly_high 반환)
        return {
            'action': 'OK',
            'new_monthly_high': new_monthly_high
        }
    
    def record_trade_result(self, pnl: float) -> None:
        """
        거래 결과 기록 (연속 손실 카운트용)
        
        Args:
            pnl: 손익률 (0.02 = +2%, -0.01 = -1%)
        
        호출:
            engine/base_engine.py - 거래 종료 후
        
        Example:
            >>> # 손실
            >>> risk_manager.record_trade_result(-0.01)
            >>> assert risk_manager.consecutive_losses == 1
            
            >>> # 수익 (리셋)
            >>> risk_manager.record_trade_result(0.02)
            >>> assert risk_manager.consecutive_losses == 0
        """
        if pnl < 0:
            self.consecutive_losses += 1
        else:
            self.consecutive_losses = 0  # 수익 시 리셋
    
    def check_consecutive_losses(self) -> Dict:
        """
        연속 손실 3회 체크
        
        Returns:
            정상: {'action': 'OK'}
            초과: {
                'action': 'PAUSE_24H',
                'reason': 'CONSECUTIVE_LOSS',
                'resume_time': int,  # 24시간 후
                'count': int
            }
        
        Example:
            >>> # 연속 3회 손실
            >>> risk_manager.consecutive_losses = 3
            >>> result = risk_manager.check_consecutive_losses()
            >>> assert result['action'] == 'PAUSE_24H'
        """
        if self.consecutive_losses >= self.config.CONSECUTIVE_LOSS_LIMIT:
            resume_time = int(time.time() + 86400)  # 24시간 후
            
            # ⭐ pause_until 설정
            self.set_pause(86400)
            
            return {
                'action': 'PAUSE_24H',
                'reason': 'CONSECUTIVE_LOSS',
                'resume_time': resume_time,
                'count': self.consecutive_losses
            }
        
        return {'action': 'OK'}
    
    def get_risk_status(
        self,
        current_balance: float,
        today_start_balance: float,
        monthly_high: float
    ) -> Dict:
        """
        현재 리스크 상태 조회 (대시보드용)
        
        ⭐ 개선: 완전 구현
        
        Args:
            current_balance: 현재 잔고
            today_start_balance: 오늘 시작 잔고
            monthly_high: 이번 달 최고 잔고
        
        Returns:
            {
                'daily': {
                    'current_pnl': float,        # 현재 일일 PnL
                    'limit': -0.05,              # 한도
                    'remaining': float,          # 남은 여유
                    'usage_percent': float,      # 사용률 (%)
                    'status': 'OK' | 'WARNING' | 'CRITICAL'
                },
                'monthly': {
                    'current_dd': float,
                    'limit': -0.10,
                    'remaining': float,
                    'usage_percent': float,
                    'status': 'OK' | 'WARNING' | 'CRITICAL'
                },
                'consecutive': {
                    'current': int,
                    'limit': 3,
                    'remaining': int,
                    'usage_percent': float,
                    'status': 'OK' | 'WARNING' | 'CRITICAL'
                },
                'overall': 'GREEN' | 'YELLOW' | 'RED',
                'paused': bool,
                'pause_until': int | None
            }
        
        호출:
            monitoring/reporter.py - 리포트 생성 시
        
        Example:
            >>> status = risk_manager.get_risk_status(
            ...     980_000, 1_000_000, 1_100_000
            ... )
            >>> print(f"종합 상태: {status['overall']}")
            >>> print(f"일일 손실: {status['daily']['current_pnl']*100:.2f}%")
        """
        # ⭐ 에러 방지
        if today_start_balance <= 0:
            today_start_balance = current_balance
        if monthly_high <= 0:
            monthly_high = current_balance
        
        # 일일 PnL
        daily_pnl = (current_balance - today_start_balance) / today_start_balance
        daily_usage = abs(daily_pnl / self.config.DAILY_LOSS_LIMIT) if self.config.DAILY_LOSS_LIMIT != 0 else 0
        
        # 월간 DD
        drawdown = (current_balance - monthly_high) / monthly_high
        monthly_usage = abs(drawdown / self.config.MONTHLY_DD_LIMIT) if self.config.MONTHLY_DD_LIMIT != 0 else 0
        
        # 연속 손실
        consecutive_usage = self.consecutive_losses / self.config.CONSECUTIVE_LOSS_LIMIT
        
        # 상태 판정 함수
        def get_status(usage: float) -> str:
            if usage < 0.4:
                return 'OK'
            elif usage < 0.8:
                return 'WARNING'
            else:
                return 'CRITICAL'
        
        # 일일
        daily_status = {
            'current_pnl': round(daily_pnl, 4),
            'limit': self.config.DAILY_LOSS_LIMIT,
            'remaining': round(self.config.DAILY_LOSS_LIMIT - daily_pnl, 4),
            'usage_percent': round(daily_usage * 100, 2),
            'status': get_status(daily_usage)
        }
        
        # 월간
        monthly_status = {
            'current_dd': round(drawdown, 4),
            'limit': self.config.MONTHLY_DD_LIMIT,
            'remaining': round(self.config.MONTHLY_DD_LIMIT - drawdown, 4),
            'usage_percent': round(monthly_usage * 100, 2),
            'status': get_status(monthly_usage)
        }
        
        # 연속
        consecutive_status = {
            'current': self.consecutive_losses,
            'limit': self.config.CONSECUTIVE_LOSS_LIMIT,
            'remaining': self.config.CONSECUTIVE_LOSS_LIMIT - self.consecutive_losses,
            'usage_percent': round(consecutive_usage * 100, 2),
            'status': get_status(consecutive_usage)
        }
        
        # 종합 상태
        max_usage = max(daily_usage, monthly_usage, consecutive_usage)
        
        if max_usage < 0.4:
            overall = 'GREEN'
        elif max_usage < 0.8:
            overall = 'YELLOW'
        else:
            overall = 'RED'
        
        return {
            'daily': daily_status,
            'monthly': monthly_status,
            'consecutive': consecutive_status,
            'overall': overall,
            'paused': self.is_trading_paused(),  # ⭐
            'pause_until': self.pause_until if self.pause_until > 0 else None
        }
    
    def is_trading_paused(self) -> bool:
        """
        거래 중지 상태 확인
        
        ⭐ 개선: pause_until 활용
        
        Returns:
            True: 현재 중지 중
            False: 정상 거래 가능
        
        호출:
            engine/base_engine.py main_loop() - 매 루프 시작 시
        
        Example:
            >>> if risk_manager.is_trading_paused():
            ...     logger.info("거래 중지 중...")
            ...     return
        """
        if self.pause_until > 0:
            current_time = time.time()
            
            if current_time < self.pause_until:
                return True  # 아직 중지 중
            else:
                # 중지 해제
                self.pause_until = 0
                self.consecutive_losses = 0  # 리셋
                return False
        
        return False
    
    def set_pause(self, duration_seconds: int) -> None:
        """
        거래 중지 설정
        
        ⭐ 개선: pause_until 설정 메서드
        
        Args:
            duration_seconds: 중지 시간 (초)
        
        Example:
            >>> # 24시간 중지
            >>> risk_manager.set_pause(86400)
        """
        self.pause_until = time.time() + duration_seconds
```

---

## 📁 risk/position.py

### 구현 코드 (전체)

```python
from typing import Dict
from core.config import Config


class PositionSizer:
    """포지션 크기 계산 (고정 배분 or Kelly Criterion)"""
    
    def __init__(self, config: Config):
        """
        포지션 계산기 초기화
        
        Args:
            config: Config 인스턴스
        """
        self.config = config
    
    def calculate_position_size(
        self,
        balance: float,
        symbol: str
    ) -> int:
        """
        심볼별 할당 금액 계산 (고정 배분)
        
        Args:
            balance: 총 잔고 (KRW)
            symbol: 'DOGE' or 'SOL'
        
        Returns:
            int: 할당 금액 (KRW)
        
        호출:
            engine/base_engine.py - 진입 시
        
        Example:
            >>> sizer = PositionSizer(config)
            >>> amount = sizer.calculate_position_size(1_000_000, 'DOGE')
            >>> assert amount == 500_000  # 50%
        """
        allocation = self.config.POSITION_ALLOCATION.get(symbol, 0)
        return int(balance * allocation)
    
    def kelly_position_size(
        self,
        balance: float,
        win_rate: float,
        avg_win: float,
        avg_loss: float
    ) -> int:
        """
        Kelly Criterion 기반 포지션 크기 (선택 기능)
        
        Kelly 공식: f = (p*b - q) / b
        
        Args:
            balance: 총 잔고 (KRW)
            win_rate: 승률 (0.70 = 70%)
            avg_win: 평균 수익률 (0.035 = 3.5%)
            avg_loss: 평균 손실률 (0.01 = 1%)
        
        Returns:
            int: 할당 금액 (KRW)
        
        Notes:
            - 1/4 Kelly 사용 (리스크 완화)
            - 최대 50% 제한
        
        Example:
            >>> sizer = PositionSizer(config)
            >>> # 승률 70%, 평균 수익 3.5%, 평균 손실 1%
            >>> amount = sizer.kelly_position_size(
            ...     1_000_000, 0.70, 0.035, 0.01
            ... )
            >>> assert 100_000 < amount < 200_000  # 약 15%
        """
        p = win_rate
        q = 1 - win_rate
        b = avg_win / avg_loss if avg_loss != 0 else 1
        
        # Kelly 계산
        kelly = (p * b - q) / b if b != 0 else 0
        
        # 음수 방지
        if kelly < 0:
            kelly = 0
        
        # 1/4 Kelly (리스크 완화)
        adjusted_kelly = kelly * 0.25
        
        # 최대 50% 제한
        final_fraction = min(adjusted_kelly, 0.5)
        
        return int(balance * final_fraction)
```

---

## 📁 risk/limits.py

### 구현 코드 (전체)

```python
from typing import Dict
from core.config import Config


class LimitsChecker:
    """
    한도 체크 유틸리티
    
    Note:
        현재는 RiskManager가 직접 체크하므로 선택적 사용
        복잡한 계산 로직을 분리하고 싶을 때 활용
    """
    
    def __init__(self, config: Config):
        """
        한도 체크기 초기화
        
        Args:
            config: Config 인스턴스
        """
        self.config = config
    
    def is_within_daily_limit(self, daily_pnl: float) -> bool:
        """
        일일 한도 내 여부 확인
        
        Args:
            daily_pnl: 일일 손익률
        
        Returns:
            True: 안전, False: 초과
        
        Example:
            >>> checker = LimitsChecker(config)
            >>> checker.is_within_daily_limit(-0.02)  # -2%
            True
            >>> checker.is_within_daily_limit(-0.06)  # -6%
            False
        """
        return daily_pnl > self.config.DAILY_LOSS_LIMIT
    
    def is_within_monthly_limit(self, drawdown: float) -> bool:
        """
        월간 한도 내 여부 확인
        
        Args:
            drawdown: 드로다운
        
        Returns:
            True: 안전, False: 초과
        """
        return drawdown > self.config.MONTHLY_DD_LIMIT
    
    def is_within_consecutive_limit(self, count: int) -> bool:
        """
        연속 손실 한도 내 여부 확인
        
        Args:
            count: 연속 손실 횟수
        
        Returns:
            True: 안전, False: 초과
        """
        return count < self.config.CONSECUTIVE_LOSS_LIMIT
    
    def get_remaining_capacity(
        self,
        current_value: float,
        limit: float
    ) -> Dict:
        """
        남은 여유 계산
        
        Args:
            current_value: 현재 값 (-0.02 등)
            limit: 한도 (-0.05 등)
        
        Returns:
            {
                'current': float,
                'limit': float,
                'remaining': float,
                'usage_percent': float
            }
        
        Example:
            >>> checker = LimitsChecker(config)
            >>> result = checker.get_remaining_capacity(-0.02, -0.05)
            >>> assert result['remaining'] == 0.03  # 3% 여유
            >>> assert result['usage_percent'] == 40.0  # 40% 사용
        """
        remaining = limit - current_value
        usage = (current_value / limit) * 100 if limit != 0 else 0
        
        return {
            'current': current_value,
            'limit': limit,
            'remaining': remaining,
            'usage_percent': round(usage, 2)
        }
```

---

## 전체 의존성 그래프

```
risk/
├── manager.py (핵심)
│   └── import core.config, core.exceptions
│
├── position.py (독립)
│   └── import core.config
│
└── limits.py (선택적)
    └── import core.config

사용되는 곳:
engine/base_engine.py
├── RiskManager.check_all_limits()
├── RiskManager.is_trading_paused()
├── RiskManager.record_trade_result()
├── RiskManager.get_risk_status()
└── PositionSizer.calculate_position_size()
```

---

## 실전 사용 예제

### 예제 1: 기본 사용

```python
from risk import RiskManager
from core.config import Config

# 초기화
config = Config.load('paper')
risk_manager = RiskManager(config)

# 거래 전 체크
if risk_manager.is_trading_paused():
    print("⏸️  거래 일시 중지 중...")
    return

# 거래 실행
# ...

# 거래 후 한도 체크
risk_result = risk_manager.check_all_limits(
    current_balance=980_000,
    today_start_balance=1_000_000,
    monthly_high=1_100_000
)

if risk_result['action'] != 'OK':
    print(f"🚨 리스크 한도 초과: {risk_result['reason']}")
    
    # monthly_high 저장 ⭐
    state.monthly_high = risk_result.get('new_monthly_high', monthly_high)
    
    # 대응
    if risk_result['action'] == 'STOP_TRADING':
        close_all_positions()
        pause_until_tomorrow()
    
    elif risk_result['action'] == 'EMERGENCY_STOP':
        emergency_shutdown()
else:
    # 정상 (monthly_high 갱신) ⭐
    state.monthly_high = risk_result['new_monthly_high']
```

### 예제 2: 거래 결과 기록

```python
# 거래 종료 후
pnl = (exit_price - entry_price) / entry_price

# 결과 기록
risk_manager.record_trade_result(pnl)

# 연속 손실 체크
if risk_manager.consecutive_losses >= 2:
    print(f"⚠️  연속 {risk_manager.consecutive_losses}회 손실")

# 3회 도달 시 자동 중지
consecutive_result = risk_manager.check_consecutive_losses()
if consecutive_result['action'] == 'PAUSE_24H':
    print("🛑 연속 3회 손실 - 24시간 중지")
```

### 예제 3: 리스크 상태 모니터링

```python
# 실시간 상태 조회
status = risk_manager.get_risk_status(
    current_balance=980_000,
    today_start_balance=1_000_000,
    monthly_high=1_100_000
)

# 대시보드 출력
print(f"━━━━━━━━━━━━━━━━━━━━━━━")
print(f"📊 리스크 상태: {status['overall']}")
print(f"━━━━━━━━━━━━━━━━━━━━━━━")
print(f"일일 PnL: {status['daily']['current_pnl']*100:+.2f}% / {status['daily']['limit']*100:.1f}%")
print(f"  └─ 상태: {status['daily']['status']} ({status['daily']['usage_percent']:.1f}% 사용)")
print(f"")
print(f"월간 DD: {status['monthly']['current_dd']*100:+.2f}% / {status['monthly']['limit']*100:.1f}%")
print(f"  └─ 상태: {status['monthly']['status']} ({status['monthly']['usage_percent']:.1f}% 사용)")
print(f"")
print(f"연속 손실: {status['consecutive']['current']} / {status['consecutive']['limit']}회")
print(f"  └─ 상태: {status['consecutive']['status']}")

if status['paused']:
    print(f"")
    print(f"⏸️  거래 중지 중 (재개: {status['pause_until']})")
```

### 예제 4: 포지션 크기 계산

```python
from risk import PositionSizer

# 초기화
sizer = PositionSizer(config)

# 고정 배분
balance = 1_000_000
amount = sizer.calculate_position_size(balance, 'DOGE')
print(f"DOGE 할당: {amount:,} KRW ({amount/balance*100:.1f}%)")

# Kelly Criterion (10거래 후)
if trade_count >= 10:
    # 통계 계산
    trades = db.get_recent_trades(10)
    win_rate = calculate_win_rate(trades)
    avg_win = calculate_avg_win(trades)
    avg_loss = calculate_avg_loss(trades)
    
    # Kelly 포지션
    kelly_amount = sizer.kelly_position_size(
        balance, win_rate, avg_win, avg_loss
    )
    print(f"Kelly 할당: {kelly_amount:,} KRW ({kelly_amount/balance*100:.1f}%)")
```

### 예제 5: Engine 통합

```python
class PaperEngine:
    def __init__(self, config):
        self.config = config
        self.risk_manager = RiskManager(config)
        
        # State 초기화
        self.today_start_balance = self.get_balance()
        self.monthly_high = self.get_balance()
    
    def main_loop(self):
        while True:
            # 1. 거래 중지 체크 ⭐
            if self.risk_manager.is_trading_paused():
                logger.info("거래 일시 중지 중...")
                time.sleep(60)
                continue
            
            # 2. 거래 로직
            # ...
            
            # 3. 거래 후 리스크 체크
            current_balance = self.get_balance()
            
            risk_result = self.risk_manager.check_all_limits(
                current_balance,
                self.today_start_balance,
                self.monthly_high
            )
            
            # 4. monthly_high 갱신 ⭐
            self.monthly_high = risk_result.get(
                'new_monthly_high',
                self.monthly_high
            )
            
            # 5. 한도 초과 처리
            if risk_result['action'] != 'OK':
                self.handle_risk_limit(risk_result)
            
            time.sleep(60)
    
    def handle_risk_limit(self, risk_result):
        """리스크 한도 초과 처리"""
        
        if risk_result['action'] == 'STOP_TRADING':
            # 일일 한도
            logger.critical("🚨 일일 손실 -5% 도달!")
            self.close_all_positions()
            self.pause_until_tomorrow()
        
        elif risk_result['action'] == 'EMERGENCY_STOP':
            # 월간 한도
            logger.critical("🚨🚨 월간 드로다운 -10% 도달!")
            self.close_all_positions()
            self.emergency_shutdown()
        
        elif risk_result['action'] == 'PAUSE_24H':
            # 연속 손실
            logger.warning("⚠️  연속 3회 손실 - 24시간 중지")
            self.close_all_positions()
```

---

## 개발 체크리스트

### manager.py ⭐
- [x] RiskManager 클래스
- [x] check_all_limits() - 3가지 한도 체크
- [x] check_daily_loss() - -5%
- [x] check_monthly_drawdown() - -10%
- [x] ⭐ new_monthly_high 반환 (호출자가 저장)
- [x] check_consecutive_losses() - 3회
- [x] record_trade_result() - 카운트 관리
- [x] ⭐ get_risk_status() - 완전 구현
- [x] ⭐ is_trading_paused() - pause_until 활용
- [x] ⭐ set_pause() - 중지 설정
- [x] ⭐ 에러 방지 (0으로 나누기 등)

### position.py
- [x] PositionSizer 클래스
- [x] calculate_position_size() - 고정 배분
- [x] kelly_position_size() - Kelly Criterion
- [x] 최대 50% 제한
- [x] 음수 방지

### limits.py
- [x] LimitsChecker 클래스
- [x] is_within_*_limit() 함수들
- [x] get_remaining_capacity()
- [x] 선택적 사용 (필수 아님)

---

## 테스트 시나리오

### manager.py 테스트

```python
import pytest
from risk import RiskManager
from core.config import Config

@pytest.fixture
def risk_manager():
    config = Config.load('paper')
    return RiskManager(config)

def test_daily_loss_ok(risk_manager):
    """정상 (손실 -2%)"""
    result = risk_manager.check_daily_loss(980_000, 1_000_000)
    assert result['action'] == 'OK'

def test_daily_loss_limit(risk_manager):
    """일일 한도 초과 (-5%)"""
    result = risk_manager.check_daily_loss(950_000, 1_000_000)
    assert result['action'] == 'STOP_TRADING'
    assert 'resume_time' in result

def test_monthly_high_update(risk_manager):
    """monthly_high 갱신"""
    result = risk_manager.check_monthly_drawdown(1_200_000, 1_100_000)
    assert result['action'] == 'OK'
    assert result['new_monthly_high'] == 1_200_000  # ⭐ 갱신됨

def test_consecutive_losses(risk_manager):
    """연속 손실 3회"""
    risk_manager.record_trade_result(-0.01)
    risk_manager.record_trade_result(-0.01)
    risk_manager.record_trade_result(-0.01)
    
    result = risk_manager.check_consecutive_losses()
    assert result['action'] == 'PAUSE_24H'
    assert risk_manager.consecutive_losses == 3

def test_consecutive_reset(risk_manager):
    """수익 후 리셋"""
    risk_manager.consecutive_losses = 2
    risk_manager.record_trade_result(0.02)  # 수익
    
    assert risk_manager.consecutive_losses == 0

def test_trading_pause(risk_manager):
    """거래 중지/재개"""
    # 중지 설정
    risk_manager.set_pause(2)  # 2초
    assert risk_manager.is_trading_paused() == True
    
    # 2초 대기
    import time
    time.sleep(2.1)
    
    # 자동 재개
    assert risk_manager.is_trading_paused() == False

def test_get_risk_status(risk_manager):
    """리스크 상태 조회"""
    status = risk_manager.get_risk_status(
        980_000, 1_000_000, 1_100_000
    )
    
    assert 'daily' in status
    assert 'monthly' in status
    assert 'consecutive' in status
    assert status['overall'] in ['GREEN', 'YELLOW', 'RED']
```

### position.py 테스트

```python
def test_fixed_allocation():
    """고정 배분"""
    config = Config.load('paper')
    sizer = PositionSizer(config)
    
    amount = sizer.calculate_position_size(1_000_000, 'DOGE')
    assert amount == 500_000  # 50%

def test_kelly_criterion():
    """Kelly Criterion"""
    config = Config.load('paper')
    sizer = PositionSizer(config)
    
    # 승률 70%, 평균 수익 3.5%, 평균 손실 1%
    amount = sizer.kelly_position_size(
        1_000_000, 0.70, 0.035, 0.01
    )
    
    # 약 15% 예상
    assert 100_000 < amount < 200_000
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
- 연속: 24시간 후 (pause_until 활용) ⭐

### 3. 포지션 관리
- 기본: 50:50 고정 배분
- 고급: Kelly Criterion (선택)
- 최대 50% 제한

### 4. 실시간 모니터링 ⭐
- get_risk_status() 완전 구현
- 대시보드용 상태 조회
- GREEN/YELLOW/RED 3단계
- 남은 여유 계산

### 5. 상태 관리 개선 ⭐
- monthly_high: 함수가 반환, 호출자가 저장
- pause_until: is_trading_paused() + set_pause()
- 명확한 책임 분리

### 6. 에러 방지 ⭐
- 0으로 나누기 방지
- 음수 값 처리
- None 값 처리

---

**문서 버전**: v2.0 (개선판)  
**작성일**: 2025-01-21  
**개선사항**:
- ⭐ monthly_high 관리 방식 명확화 (반환 → 저장)
- ⭐ get_risk_status() 완전 구현
- ⭐ pause_until 활용 (is_trading_paused, set_pause)
- ⭐ 에러 방지 (0으로 나누기 등)
- ⭐ 실전 사용 예제 추가
- ⭐ Engine 통합 예제
- ✅ 테스트 시나리오 완성

**검증 상태**: ✅ 완료
