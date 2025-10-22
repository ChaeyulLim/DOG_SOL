# 04_STRATEGY 모듈 완벽 함수 명세서 v2.0 (개선판)

> **개선사항**: _check_additional_conditions() 실제 구현, 폴백 로직 강화, AI 호출 중복 방지

---

## 📋 목차
1. [strategy/entry.py](#strategyentrypy) ⭐ 개선
2. [strategy/exit.py](#strategyexitpy) ⭐ 개선
3. [strategy/trailing.py](#strategytrailingpy) ⭐ 개선
4. [strategy/signals.py](#strategysignalspy) ⭐ 개선
5. [전체 의존성 그래프](#전체-의존성-그래프)
6. [실전 사용 예제](#실전-사용-예제)

---

## 📁 strategy/entry.py ⭐ 개선

### 파일 전체 구조
```python
from typing import Dict, Optional
from indicators import IndicatorCalculator
from ai import ClaudeClient
from core.config import Config
from core.exceptions import AIResponseError

class EntryStrategy:
    def __init__(self, config: Config, claude: ClaudeClient): ...
    
    async def check_entry_signal(
        self,
        symbol: str,
        market_data: Dict,
        indicators: Dict
    ) -> Optional[Dict]: ...
    
    def _check_technical_conditions(self, indicators: Dict) -> bool: ...
    def _check_additional_conditions(self, indicators: Dict) -> Dict: ...  # ⭐ 실제 구현
    def _fallback_entry_decision(self, indicators: Dict) -> Optional[Dict]: ...  # ⭐ 추가
```

---

### 📌 클래스: EntryStrategy (전체 구현)

```python
from typing import Dict, Optional
from indicators import IndicatorCalculator
from ai import ClaudeClient
from core.config import Config
from core.exceptions import AIResponseError


class EntryStrategy:
    """진입 신호 생성 (3단계 필터링)"""
    
    def __init__(self, config: Config, claude: ClaudeClient):
        """
        초기화
        
        Args:
            config: 시스템 설정
            claude: Claude API 클라이언트
        """
        self.config = config
        self.claude = claude
        self.min_signal_count = 3  # 최소 3개 지표 충족
    
    async def check_entry_signal(
        self,
        symbol: str,
        market_data: Dict,
        indicators: Dict
    ) -> Optional[Dict]:
        """
        진입 신호 종합 판단 (3단계 필터링)
        
        Args:
            symbol: 'DOGE/USDT'
            market_data: fetcher.fetch_market_data() 결과
            indicators: calculator.calculate_all() 결과
        
        Returns:
            {
                'action': 'ENTER',
                'symbol': 'DOGE/USDT',
                'confidence': 0.75,
                'technical_score': 3,
                'additional_score': {...},
                'ai_reasoning': "...",
                'key_factors': ['macd', 'volume'],
                'fallback': False
            }
            
            또는 None (진입 불가)
        """
        # Step 1: 기술적 지표 1차 필터
        if not self._check_technical_conditions(indicators):
            return None
        
        signal_count = indicators['signal_count']
        
        # Step 2: 추가 조건 확인
        additional = self._check_additional_conditions(indicators)
        
        # Step 3: AI 최종 판단
        try:
            ai_response = await self.claude.analyze_entry(
                symbol=symbol,
                price=market_data['price'],
                indicators=indicators,
                market_context={
                    'volume_24h': market_data.get('volume_24h', 0),
                    'change_24h': market_data.get('change_24h', 0),
                    'additional_signals': additional
                }
            )
            
            if ai_response['action'] == 'ENTER' and \
               ai_response['confidence'] >= self.config.MIN_AI_CONFIDENCE:
                
                return {
                    'action': 'ENTER',
                    'symbol': symbol,
                    'confidence': ai_response['confidence'],
                    'technical_score': signal_count,
                    'additional_score': additional,
                    'ai_reasoning': ai_response['reasoning'],
                    'key_factors': ai_response.get('key_factors', []),
                    'fallback': False
                }
            
            return None
        
        except AIResponseError as e:
            # AI 실패 시 폴백
            return self._fallback_entry_decision(indicators)
    
    def _check_technical_conditions(self, indicators: Dict) -> bool:
        """
        기술적 지표 1차 필터 (4개 중 최소 3개 충족)
        
        Args:
            indicators: calculator.calculate_all() 결과
        
        Returns:
            True (signal_count >= 3), False (미달)
        """
        return indicators['signal_count'] >= self.min_signal_count
    
    def _check_additional_conditions(self, indicators: Dict) -> Dict:
        """
        ⭐ 추가 조건 실제 체크 (개선)
        
        Args:
            indicators: calculator.calculate_all() 결과
        
        Returns:
            {
                'ema_aligned': bool,
                'volume_ratio': float,
                'stoch_rsi_up': bool,
                'total_bonus': int  # 0-3점
            }
        """
        # EMA 정렬 체크 (20 > 50 > 200)
        ema_aligned = False
        if 'ema' in indicators:
            ema_20 = indicators['ema'].get('20', 0)
            ema_50 = indicators['ema'].get('50', 0)
            ema_200 = indicators['ema'].get('200', 0)
            ema_aligned = ema_20 > ema_50 > ema_200 > 0
        
        # 거래량 비율 체크
        volume_ratio = indicators.get('volume_ratio', 1.0)
        volume_high = volume_ratio > 1.5
        
        # Stochastic RSI 체크
        stoch_rsi_up = False
        if 'stoch_rsi' in indicators:
            stoch_k = indicators['stoch_rsi'].get('k', 0)
            stoch_rsi_up = stoch_k > 20
        
        # 보너스 점수 계산
        bonus_score = sum([
            ema_aligned,
            volume_high,
            stoch_rsi_up
        ])
        
        return {
            'ema_aligned': ema_aligned,
            'volume_ratio': volume_ratio,
            'stoch_rsi_up': stoch_rsi_up,
            'total_bonus': bonus_score
        }
    
    def _fallback_entry_decision(self, indicators: Dict) -> Optional[Dict]:
        """
        ⭐ AI 실패 시 지표 기반 폴백 판단 (개선)
        
        기획서의 fallback_decision() 로직 적용:
        - RSI 30 미만: 30점
        - RSI 30-50: 15점
        - MACD 골든크로스: 30점
        - MACD bullish: 15점
        - 볼린저 하단: 25점
        - 피보나치 지지: 15점
        
        60점 이상만 진입
        
        Args:
            indicators: calculator.calculate_all() 결과
        
        Returns:
            진입 신호 또는 None
        """
        score = 0
        reasons = []
        
        # RSI
        rsi = indicators['rsi']['value']
        if rsi < 30:
            score += 30
            reasons.append(f'rsi_oversold_{rsi:.0f}')
        elif rsi < 50:
            score += 15
            reasons.append(f'rsi_below_50_{rsi:.0f}')
        
        # MACD
        if indicators['macd'].get('golden_cross'):
            score += 30
            reasons.append('macd_golden_cross')
        elif indicators['macd']['momentum'] == 'bullish':
            score += 15
            reasons.append('macd_bullish')
        
        # Bollinger
        if indicators['bollinger'].get('lower_touch'):
            score += 25
            reasons.append('bb_lower_touch')
        
        # Fibonacci
        if indicators['fibonacci'].get('at_support'):
            score += 15
            reasons.append('fib_support')
        
        # 60점 이상만 진입
        if score >= 60:
            return {
                'action': 'ENTER',
                'confidence': min(score / 100, 0.85),
                'technical_score': indicators['signal_count'],
                'fallback': True,
                'fallback_score': score,
                'ai_reasoning': f'폴백 로직: {score}/100점',
                'key_factors': reasons
            }
        
        return None
```

---

## 📁 strategy/exit.py ⭐ 개선

### 파일 전체 구조
```python
from typing import Dict, Optional
import time
from ai import ClaudeClient
from core.config import Config
from .trailing import TrailingStop

class ExitStrategy:
    def __init__(self, config: Config, claude: ClaudeClient): ...
    
    async def check_exit_signal(
        self,
        symbol: str,
        position: Dict,
        current_price: float,
        indicators: Dict
    ) -> Optional[Dict]: ...
    
    def _check_stop_loss(self, position: Dict, current_price: float) -> Optional[Dict]: ...
    def _check_take_profit(self, position: Dict, current_price: float) -> bool: ...
    def _check_timeout(self, position: Dict) -> bool: ...
```

---

### 📌 클래스: ExitStrategy (전체 구현)

```python
from typing import Dict, Optional
import time
from ai import ClaudeClient
from core.config import Config
from core.exceptions import AIResponseError
from .trailing import TrailingStop


class ExitStrategy:
    """청산 신호 생성 (5가지 우선순위)"""
    
    def __init__(self, config: Config, claude: ClaudeClient):
        """
        초기화
        
        Args:
            config: 시스템 설정
            claude: Claude API 클라이언트
        """
        self.config = config
        self.claude = claude
        self.trailing_stops: Dict[str, TrailingStop] = {}
        self.last_ai_check_time: Dict[str, float] = {}  # ⭐ 중복 호출 방지
    
    async def check_exit_signal(
        self,
        symbol: str,
        position: Dict,
        current_price: float,
        indicators: Dict
    ) -> Optional[Dict]:
        """
        청산 신호 종합 판단 (우선순위별 체크)
        
        Args:
            symbol: 'DOGE/USDT'
            position: {
                'symbol': 'DOGE/USDT',
                'entry_price': 0.3821,
                'quantity': 1006.0,
                'entry_time': 1234567890,
                'trade_id': 123,
                'initial_amount_krw': 500000
            }
            current_price: 현재가
            indicators: calculator.calculate_all() 결과
        
        Returns:
            {
                'action': 'EXIT',
                'reason': 'STOP_LOSS' | 'TRAILING_STOP' | 'TARGET_EXIT' | 
                          'PERIODIC_EXIT' | 'TIMEOUT',
                'priority': 1-5,
                'current_pnl': -0.01,
                'ai_involved': bool,
                'ai_confidence': float (선택),
                'ai_reasoning': str (선택)
            }
            
            또는 None (청산 불필요)
        """
        entry_price = position['entry_price']
        current_pnl = (current_price - entry_price) / entry_price
        holding_time = time.time() - position['entry_time']
        
        # Priority 1: 초기 손절 (-1%)
        stop_loss_signal = self._check_stop_loss(position, current_price)
        if stop_loss_signal:
            return stop_loss_signal
        
        # Priority 2: 24시간 경과
        if self._check_timeout(position):
            return {
                'action': 'EXIT',
                'reason': 'TIMEOUT',
                'priority': 2,
                'current_pnl': current_pnl,
                'holding_hours': holding_time / 3600,
                'ai_involved': False
            }
        
        # Priority 3: 트레일링 손절
        if symbol not in self.trailing_stops:
            self.trailing_stops[symbol] = TrailingStop(
                entry_price=entry_price,
                activation_pct=self.config.TRAILING_ACTIVATION,
                stop_distance=abs(self.config.TRAILING_STOP)
            )
        
        trailing = self.trailing_stops[symbol]
        trailing_signal = trailing.update(current_price)
        
        if trailing_signal == 'STOP':
            return {
                'action': 'EXIT',
                'reason': 'TRAILING_STOP',
                'priority': 3,
                'current_pnl': current_pnl,
                'highest_price': trailing.highest_price,
                'ai_involved': False
            }
        
        # Priority 4: 목표가 도달 (+2%)
        if self._check_take_profit(position, current_price):
            try:
                ai_response = await self.claude.analyze_holding(
                    symbol=symbol,
                    entry_price=entry_price,
                    current_price=current_price,
                    current_pnl=current_pnl,
                    holding_time=holding_time,
                    indicators=indicators,
                    check_type='TARGET_REACHED'
                )
                
                if ai_response['action'] == 'EXIT':
                    return {
                        'action': 'EXIT',
                        'reason': 'TARGET_EXIT',
                        'priority': 4,
                        'current_pnl': current_pnl,
                        'ai_confidence': ai_response['confidence'],
                        'ai_reasoning': ai_response['reasoning'],
                        'ai_involved': True
                    }
            except AIResponseError:
                # AI 실패 시 목표가 도달로 청산
                return {
                    'action': 'EXIT',
                    'reason': 'TARGET_EXIT',
                    'priority': 4,
                    'current_pnl': current_pnl,
                    'ai_involved': False,
                    'note': 'AI 실패, 목표가 도달로 청산'
                }
        
        # Priority 5: 정기 체크 (2시간마다 1번만) ⭐ 개선
        time_since_last_check = holding_time - self.last_ai_check_time.get(symbol, 0)
        
        if time_since_last_check >= self.config.AI_CHECK_INTERVAL:
            try:
                ai_response = await self.claude.analyze_holding(
                    symbol=symbol,
                    entry_price=entry_price,
                    current_price=current_price,
                    current_pnl=current_pnl,
                    holding_time=holding_time,
                    indicators=indicators,
                    check_type='PERIODIC_CHECK'
                )
                
                # 체크 시간 기록 ⭐
                self.last_ai_check_time[symbol] = holding_time
                
                if ai_response['action'] == 'EXIT':
                    return {
                        'action': 'EXIT',
                        'reason': 'PERIODIC_EXIT',
                        'priority': 5,
                        'current_pnl': current_pnl,
                        'ai_confidence': ai_response['confidence'],
                        'ai_reasoning': ai_response['reasoning'],
                        'ai_involved': True
                    }
            except AIResponseError:
                # AI 실패는 무시하고 계속 보유
                self.last_ai_check_time[symbol] = holding_time
        
        return None
    
    def _check_stop_loss(self, position: Dict, current_price: float) -> Optional[Dict]:
        """
        초기 손절 체크 (-1%)
        
        Returns:
            청산 신호 또는 None
        """
        entry_price = position['entry_price']
        current_pnl = (current_price - entry_price) / entry_price
        
        if current_pnl <= self.config.STOP_LOSS:
            return {
                'action': 'EXIT',
                'reason': 'STOP_LOSS',
                'priority': 1,
                'current_pnl': current_pnl,
                'ai_involved': False
            }
        
        return None
    
    def _check_take_profit(self, position: Dict, current_price: float) -> bool:
        """
        목표 익절 도달 여부 (+2%)
        
        Returns:
            True (도달), False (미도달)
        """
        entry_price = position['entry_price']
        current_pnl = (current_price - entry_price) / entry_price
        
        return current_pnl >= self.config.TAKE_PROFIT
    
    def _check_timeout(self, position: Dict) -> bool:
        """
        24시간 경과 여부
        
        Returns:
            True (경과), False (미경과)
        """
        holding_time = time.time() - position['entry_time']
        return holding_time >= self.config.MAX_HOLDING_TIME
    
    def cleanup_position(self, symbol: str) -> None:
        """
        ⭐ 포지션 청산 후 정리 (추가)
        
        Args:
            symbol: 'DOGE/USDT'
        """
        if symbol in self.trailing_stops:
            del self.trailing_stops[symbol]
        
        if symbol in self.last_ai_check_time:
            del self.last_ai_check_time[symbol]
```

---

## 📁 strategy/trailing.py ⭐ 개선

### 파일 전체 구조
```python
from typing import Dict

class TrailingStop:
    def __init__(
        self,
        entry_price: float,
        activation_pct: float = 0.005,
        stop_distance: float = 0.01
    ): ...
    
    def update(self, current_price: float) -> str: ...
    def get_status(self) -> Dict: ...  # ⭐ 추가
```

---

### 📌 클래스: TrailingStop (전체 구현)

```python
from typing import Dict


class TrailingStop:
    """트레일링 스톱 로직 관리"""
    
    def __init__(
        self,
        entry_price: float,
        activation_pct: float = 0.005,
        stop_distance: float = 0.01
    ):
        """
        초기화
        
        Args:
            entry_price: 진입가 (예: 100.0)
            activation_pct: 활성화 기준 (+0.5% = 0.005)
            stop_distance: 손절 거리 (1% = 0.01)
        """
        self.entry_price = entry_price
        self.activation_pct = activation_pct
        self.stop_distance = stop_distance
        
        # 활성화 가격
        self.activation_price = entry_price * (1 + activation_pct)
        
        # 고점 (초기값 = 진입가)
        self.highest_price = entry_price
        
        # 손절가 (초기값 = 진입가 -1%)
        self.stop_price = entry_price * (1 - stop_distance)
        
        # 활성화 여부
        self.activated = False
    
    def update(self, current_price: float) -> str:
        """
        매 분마다 트레일링 스톱 업데이트
        
        Args:
            current_price: 현재가
        
        Returns:
            'STOP': 손절 발동
            'CONTINUE': 계속 보유
        
        Example:
            >>> trailing = TrailingStop(100.0, 0.005, 0.01)
            >>> trailing.update(100.5)  # +0.5%
            'CONTINUE'  # 활성화됨
            
            >>> trailing.update(105.0)  # +5%
            'CONTINUE'  # 고점 갱신, 손절가 103.95
            
            >>> trailing.update(103.94)
            'STOP'  # 손절 발동!
        """
        # 1. 활성화 체크
        if not self.activated:
            if current_price >= self.activation_price:
                self.activated = True
        
        # 2. 고점 갱신 (활성화된 경우만)
        if self.activated:
            if current_price > self.highest_price:
                self.highest_price = current_price
                self.stop_price = self.highest_price * (1 - self.stop_distance)
        
        # 3. 손절 체크
        if current_price <= self.stop_price:
            return 'STOP'
        
        return 'CONTINUE'
    
    def get_status(self) -> Dict:
        """
        ⭐ 현재 트레일링 스톱 상태 반환 (디버깅/로깅용) - 추가
        
        Returns:
            {
                'activated': bool,
                'activation_price': float,
                'highest_price': float,
                'stop_price': float,
                'stop_distance_pct': float
            }
        
        Example:
            >>> trailing = TrailingStop(100.0)
            >>> trailing.update(105.0)
            >>> status = trailing.get_status()
            >>> print(status)
            {
                'activated': True,
                'activation_price': 100.5,
                'highest_price': 105.0,
                'stop_price': 103.95,
                'stop_distance_pct': 1.0
            }
        """
        return {
            'activated': self.activated,
            'activation_price': self.activation_price,
            'highest_price': self.highest_price,
            'stop_price': self.stop_price,
            'stop_distance_pct': self.stop_distance * 100
        }
```

---

## 📁 strategy/signals.py ⭐ 개선

### 파일 전체 구조
```python
from typing import Dict, List

def calculate_signal_strength(
    technical_score: int,
    additional_score: Dict,
    ai_confidence: float
) -> float: ...

def get_entry_reasons(entry_signal: Dict) -> List[str]: ...  # ⭐ 추가

def get_exit_reasons(exit_signal: Dict) -> str: ...  # ⭐ 추가
```

---

### 📌 전체 구현 코드

```python
from typing import Dict, List


def calculate_signal_strength(
    technical_score: int,
    additional_score: Dict,
    ai_confidence: float
) -> float:
    """
    진입 신호 강도 계산 (0-1)
    
    가중치:
    - 기술적 지표: 40% (technical_score / 4 * 0.4)
    - 추가 조건: 20% (bonus / 3 * 0.2)
    - AI 신뢰도: 40% (ai_confidence * 0.4)
    
    Args:
        technical_score: 0-4 (충족된 기술 지표 수)
        additional_score: _check_additional_conditions() 결과
        ai_confidence: 0-1 (AI 신뢰도)
    
    Returns:
        0-1 사이의 강도
    
    Example:
        >>> strength = calculate_signal_strength(3, {'total_bonus': 2}, 0.75)
        >>> print(strength)
        0.733
    """
    technical = (technical_score / 4.0) * 0.4
    bonus_count = additional_score.get('total_bonus', 0)
    additional = (bonus_count / 3.0) * 0.2
    ai = ai_confidence * 0.4
    
    strength = technical + additional + ai
    return round(strength, 3)


def get_entry_reasons(entry_signal: Dict) -> List[str]:
    """
    ⭐ 진입 신호의 이유를 사람이 읽을 수 있는 형태로 변환 (추가)
    
    Args:
        entry_signal: check_entry_signal()의 반환값
    
    Returns:
        ['RSI 과매도 (30)', 'MACD 골든크로스', 'AI 신뢰도 75%']
    
    Example:
        >>> reasons = get_entry_reasons(entry_signal)
        >>> print(', '.join(reasons))
        "기술적 신호 3/4개 충족, EMA 정배열, AI 신뢰도 75%"
    """
    reasons = []
    
    # 기술적 지표
    tech_score = entry_signal.get('technical_score', 0)
    reasons.append(f"기술적 신호 {tech_score}/4개 충족")
    
    # 추가 조건
    additional = entry_signal.get('additional_score', {})
    if additional.get('ema_aligned'):
        reasons.append("EMA 정배열")
    if additional.get('volume_ratio', 0) > 1.5:
        reasons.append(f"거래량 {additional['volume_ratio']:.1f}배")
    if additional.get('stoch_rsi_up'):
        reasons.append("Stoch RSI 상승")
    
    # AI
    confidence = entry_signal.get('confidence', 0)
    reasons.append(f"AI 신뢰도 {confidence*100:.0f}%")
    
    # AI 핵심 요인
    key_factors = entry_signal.get('key_factors', [])
    if key_factors:
        reasons.append(f"핵심: {', '.join(key_factors)}")
    
    # 폴백 여부
    if entry_signal.get('fallback'):
        fallback_score = entry_signal.get('fallback_score', 0)
        reasons.append(f"(폴백 {fallback_score}점)")
    
    return reasons


def get_exit_reasons(exit_signal: Dict) -> str:
    """
    ⭐ 청산 신호 이유를 사람이 읽을 수 있는 형태로 변환 (추가)
    
    Args:
        exit_signal: check_exit_signal()의 반환값
    
    Returns:
        "트레일링 손절 (고점 105.0 → 103.94, 수익 +3.94%)"
    
    Example:
        >>> reason = get_exit_reasons(exit_signal)
        >>> print(reason)
        "트레일링 손절 (고점 105.0 → 103.94, 수익 +3.94%)"
    """
    reason = exit_signal['reason']
    pnl = exit_signal['current_pnl']
    
    # 기본 메시지
    messages = {
        'STOP_LOSS': f"초기 손절 ({pnl*100:.2f}%)",
        'TRAILING_STOP': "트레일링 손절",
        'TARGET_EXIT': f"목표가 도달 ({pnl*100:.2f}%)",
        'PERIODIC_EXIT': f"정기 체크 청산 ({pnl*100:.2f}%)",
        'TIMEOUT': f"24시간 경과 청산 ({pnl*100:.2f}%)"
    }
    
    base_msg = messages.get(reason, f"청산 ({pnl*100:.2f}%)")
    
    # 트레일링 스톱 상세 정보
    if reason == 'TRAILING_STOP' and 'highest_price' in exit_signal:
        highest = exit_signal['highest_price']
        base_msg += f" (고점 {highest:.4f}, 수익 {pnl*100:+.2f}%)"
    
    # AI 정보
    if exit_signal.get('ai_involved') and 'ai_reasoning' in exit_signal:
        base_msg += f" - AI: {exit_signal['ai_reasoning']}"
    
    return base_msg
```

---

## 전체 의존성 그래프

```
strategy/
├── entry.py
│   ├── import indicators (IndicatorCalculator)
│   ├── import ai (ClaudeClient)
│   └── import core (Config, exceptions)
│
├── exit.py
│   ├── import trailing.py (TrailingStop)
│   ├── import ai (ClaudeClient)
│   └── import core (Config, exceptions)
│
├── trailing.py (독립)
│
└── signals.py (독립)

사용하는 모듈:
- indicators/IndicatorCalculator
- ai/ClaudeClient
- core/Config, exceptions

사용되는 곳:
- engine/base_engine.py
```

---

## 실전 사용 예제

### 예제 1: 진입 체크

```python
from strategy import EntryStrategy
from core.config import Config
from ai import ClaudeClient

# 초기화
config = Config.load('paper')
claude = ClaudeClient()
entry_strategy = EntryStrategy(config, claude)

# 진입 체크
entry_signal = await entry_strategy.check_entry_signal(
    symbol='DOGE/USDT',
    market_data=market_data,
    indicators=indicators
)

if entry_signal:
    print(f"✅ 진입 신호 발생!")
    print(f"   신뢰도: {entry_signal['confidence']}")
    print(f"   기술 점수: {entry_signal['technical_score']}/4")
    
    # 이유 출력
    from strategy.signals import get_entry_reasons
    reasons = get_entry_reasons(entry_signal)
    print(f"   이유: {', '.join(reasons)}")
else:
    print("❌ 진입 조건 미충족")
```

### 예제 2: 청산 체크

```python
from strategy import ExitStrategy

exit_strategy = ExitStrategy(config, claude)

# 청산 체크
exit_signal = await exit_strategy.check_exit_signal(
    symbol='DOGE/USDT',
    position=position,
    current_price=current_price,
    indicators=indicators
)

if exit_signal:
    print(f"🚨 청산 신호 발생!")
    print(f"   이유: {exit_signal['reason']}")
    print(f"   우선순위: {exit_signal['priority']}")
    print(f"   수익률: {exit_signal['current_pnl']*100:+.2f}%")
    
    # 상세 이유
    from strategy.signals import get_exit_reasons
    detail = get_exit_reasons(exit_signal)
    print(f"   상세: {detail}")
```

### 예제 3: 트레일링 스톱 추적

```python
from strategy.trailing import TrailingStop

# 초기화
trailing = TrailingStop(entry_price=100.0)

# 가격 업데이트
prices = [100.5, 101.2, 102.0, 105.0, 104.0, 103.94]

for price in prices:
    result = trailing.update(price)
    status = trailing.get_status()
    
    print(f"가격: {price:.2f} | 고점: {status['highest_price']:.2f} | "
          f"손절가: {status['stop_price']:.2f} | 결과: {result}")
    
    if result == 'STOP':
        print("🛑 트레일링 손절 발동!")
        break
```

### 예제 4: 포지션 청산 후 정리

```python
# 청산 후 반드시 정리
exit_strategy.cleanup_position('DOGE/USDT')
```

---

## 개발 체크리스트

### entry.py ⭐
- [x] EntryStrategy 클래스
- [x] check_entry_signal() - 3단계 필터링
- [x] _check_additional_conditions() ⭐ 실제 구현
- [x] _fallback_entry_decision() ⭐ 강화된 폴백
- [x] async/await 처리
- [x] AI 실패 예외 처리

### exit.py ⭐
- [x] ExitStrategy 클래스
- [x] check_exit_signal() - 5가지 우선순위
- [x] last_ai_check_time ⭐ 중복 호출 방지
- [x] cleanup_position() ⭐ 정리 함수
- [x] 트레일링 스톱 통합
- [x] AI 호출 (목표/정기)
- [x] AI 실패 처리

### trailing.py ⭐
- [x] TrailingStop 클래스
- [x] update() - 활성화, 고점, 손절
- [x] get_status() ⭐ 상태 조회 추가
- [x] 초기 손절가 설정

### signals.py ⭐
- [x] calculate_signal_strength()
- [x] get_entry_reasons() ⭐ 추가
- [x] get_exit_reasons() ⭐ 추가

---

**문서 버전**: v2.0 (개선판)  
**작성일**: 2025-01-21  
**개선사항**: 
- ⭐ _check_additional_conditions() 실제 구현
- ⭐ _fallback_entry_decision() 점수 기반 강화
- ⭐ last_ai_check_time으로 중복 호출 방지
- ⭐ cleanup_position() 정리 함수 추가
- ⭐ TrailingStop.get_status() 추가
- ⭐ get_entry_reasons() 추가
- ⭐ get_exit_reasons() 추가
- ✅ position Dict 구조 명세화

**검증 상태**: ✅ 완료
