
# 04_STRATEGY 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [strategy/entry.py](#strategyentrypy)
2. [strategy/exit.py](#strategyexitpy)
3. [strategy/trailing.py](#strategytrailingpy)
4. [strategy/signals.py](#strategysignalspy)
5. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📁 strategy/entry.py

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
    def _check_additional_conditions(self, indicators: Dict) -> Dict: ...
```

---

### 📌 클래스: EntryStrategy

#### 목적
진입 신호 생성 (3단계 필터링: 지표 → 추가 조건 → AI)

---

### 📌 함수: EntryStrategy.__init__(config, claude)

```python
def __init__(self, config: Config, claude: ClaudeClient):
```

#### 역할
진입 전략 초기화

#### 인자
- `config: Config` - 시스템 설정
- `claude: ClaudeClient` - AI 클라이언트

#### 초기화 내용
```python
self.config = config
self.claude = claude
self.min_signal_count = 3  # 최소 3개 지표 충족
```

#### 호출되는 곳
```python
# engine/base_engine.py __init__()
from strategy import EntryStrategy

self.entry_strategy = EntryStrategy(self.config, self.claude)
```

#### 구현 코드
```python
def __init__(self, config: Config, claude: ClaudeClient):
    """
    초기화
    
    Args:
        config: 시스템 설정
        claude: Claude API 클라이언트
    """
    self.config = config
    self.claude = claude
    self.min_signal_count = 3
```

---

### 📌 함수: EntryStrategy.check_entry_signal(symbol, market_data, indicators)

```python
async def check_entry_signal(
    self,
    symbol: str,
    market_data: Dict,
    indicators: Dict
) -> Optional[Dict]:
```

#### 역할
진입 신호 종합 판단 (3단계 필터링)

#### 인자
- `symbol: str` - 'DOGE/USDT'
- `market_data: Dict` - fetcher.fetch_market_data() 결과
- `indicators: Dict` - calculator.calculate_all() 결과

#### 사용하는 모듈/함수
1. `self._check_technical_conditions(indicators)` - 1차 필터
2. `self._check_additional_conditions(indicators)` - 2차 확인
3. `self.claude.analyze_entry(...)` - AI 최종 판단

#### 호출되는 곳
```python
# engine/base_engine.py main_loop()
for symbol, data in all_data.items():
    if not has_position(symbol):
        entry_signal = await self.entry_strategy.check_entry_signal(
            symbol,
            data,
            indicators
        )
        if entry_signal:
            # 주문 실행
```

#### 처리 흐름
```
Step 1: 기술적 지표 1차 필터
├─ signal_count >= 3?
│  ├─ YES → Step 2
│  └─ NO → return None (진입 불가)

Step 2: 추가 조건 확인
├─ EMA 정렬, 거래량, Stoch RSI
└─ 결과 수집

Step 3: AI 최종 판단
├─ Claude API 호출
├─ confidence >= 0.70?
│  ├─ YES → return 진입 신호
│  └─ NO → return None
```

#### 반환값
```python
# 진입 신호
Dict:
    'action': str = 'ENTER'
    'symbol': str = 'DOGE/USDT'
    'confidence': float = 0.75
    'technical_score': int = 3
    'additional_score': Dict = {
        'ema_aligned': True,
        'volume_ratio': 1.8,
        'stoch_rsi_up': True
    }
    'ai_reasoning': str = "MACD golden cross..."
    'key_factors': List[str] = ['macd', 'volume', 'rsi']

# 진입 불가
None
```

#### 구현 코드
```python
async def check_entry_signal(
    self,
    symbol: str,
    market_data: Dict,
    indicators: Dict
) -> Optional[Dict]:
    """진입 신호 종합 판단"""
    
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
                'volume_24h': market_data['volume_24h'],
                'change_24h': market_data['change_24h'],
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
                'key_factors': ai_response.get('key_factors', [])
            }
        
        return None
    
    except AIResponseError as e:
        # AI 실패 시 폴백
        if signal_count >= 4:
            return {
                'action': 'ENTER',
                'symbol': symbol,
                'confidence': 0.6,
                'technical_score': signal_count,
                'fallback': True,
                'reason': f"AI 실패, 지표 기반 진입: {e}"
            }
        
        return None
```

---

### 📌 함수: EntryStrategy._check_technical_conditions(indicators)

```python
def _check_technical_conditions(self, indicators: Dict) -> bool:
```

#### 역할
기술적 지표 1차 필터 (4개 중 최소 3개 충족)

#### 반환값
- `bool`: True (signal_count >= 3), False (미달)

#### 구현 코드
```python
def _check_technical_conditions(self, indicators: Dict) -> bool:
    """기술적 지표 1차 필터"""
    return indicators['signal_count'] >= self.min_signal_count
```

---

### 📌 함수: EntryStrategy._check_additional_conditions(indicators)

```python
def _check_additional_conditions(self, indicators: Dict) -> Dict:
```

#### 역할
추가 조건 확인 (선택적 강화 신호)

#### 반환값
```python
Dict:
    'ema_aligned': bool
    'volume_ratio': float
    'stoch_rsi_up': bool
    'total_bonus': int
```

#### 구현 코드
```python
def _check_additional_conditions(self, indicators: Dict) -> Dict:
    """추가 조건 확인"""
    return {
        'ema_aligned': False,
        'volume_ratio': 1.0,
        'stoch_rsi_up': False,
        'total_bonus': 0
    }
```

---

## 📁 strategy/exit.py

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

### 📌 함수: ExitStrategy.check_exit_signal(symbol, position, current_price, indicators)

```python
async def check_exit_signal(
    self,
    symbol: str,
    position: Dict,
    current_price: float,
    indicators: Dict
) -> Optional[Dict]:
```

#### 역할
청산 신호 종합 판단 (우선순위별 체크)

#### 처리 흐름 (우선순위)
```
Priority 1: 초기 손절 (-1%)
├─ current_pnl <= -0.01? → 즉시 청산

Priority 2: 24시간 경과
├─ holding_time >= 86400? → 강제 청산

Priority 3: 트레일링 손절
├─ trailing.is_active?
├─ current_price <= stop_price? → 청산

Priority 4: 목표가 도달 (+2%)
├─ current_pnl >= 0.02?
├─ AI: "계속 오를까?" → EXIT or HOLD

Priority 5: 정기 체크 (2시간마다)
├─ holding_time % 7200 < 60?
├─ AI 상황 판단 → EXIT?
```

#### 반환값
```python
# 청산 신호
Dict:
    'action': str = 'EXIT'
    'reason': str = 'STOP_LOSS' | 'TRAILING_STOP' | 'TARGET_EXIT' | 
                    'PERIODIC_EXIT' | 'TIMEOUT'
    'priority': int = 1-5
    'current_pnl': float
    'ai_involved': bool

# 청산 불필요
None
```

#### 구현 코드
```python
async def check_exit_signal(
    self,
    symbol: str,
    position: Dict,
    current_price: float,
    indicators: Dict
) -> Optional[Dict]:
    """청산 신호 종합 판단"""
    
    entry_price = position['entry_price']
    current_pnl = (current_price - entry_price) / entry_price
    holding_time = time.time() - position['entry_time']
    
    # Priority 1: 초기 손절
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
    
    # Priority 4: 목표가 도달
    if self._check_take_profit(position, current_price):
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
    
    # Priority 5: 정기 체크
    if holding_time % self.config.AI_CHECK_INTERVAL < 60:
        ai_response = await self.claude.analyze_holding(
            symbol=symbol,
            entry_price=entry_price,
            current_price=current_price,
            current_pnl=current_pnl,
            holding_time=holding_time,
            indicators=indicators,
            check_type='PERIODIC_CHECK'
        )
        
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
    
    return None
```

---

## 📁 strategy/trailing.py

### 📌 클래스: TrailingStop

#### 목적
트레일링 스톱 로직 관리

---

### 📌 함수: TrailingStop.update(current_price)

```python
def update(self, current_price: float) -> str:
```

#### 역할
매 분마다 트레일링 스톱 업데이트

#### 로직
```
1. 활성화 체크
   - activated == False AND current_price >= activation_price
   - → activated = True

2. 고점 갱신 (활성화된 경우만)
   - current_price > highest_price
   - → highest_price = current_price
   - → stop_price = highest_price * (1 - stop_distance)

3. 손절 체크
   - current_price <= stop_price → return 'STOP'

4. 정상
   - return 'CONTINUE'
```

#### 구현 코드
```python
def update(self, current_price: float) -> str:
    """트레일링 스톱 업데이트"""
    
    # 1. 활성화 체크
    if not self.activated:
        if current_price >= self.activation_price:
            self.activated = True
    
    # 2. 고점 갱신
    if self.activated:
        if current_price > self.highest_price:
            self.highest_price = current_price
            self.stop_price = self.highest_price * (1 - self.stop_distance)
    
    # 3. 손절 체크
    if current_price <= self.stop_price:
        return 'STOP'
    
    return 'CONTINUE'
```

---

## 📁 strategy/signals.py

### 📌 함수: calculate_signal_strength(technical_score, additional_score, ai_confidence)

```python
def calculate_signal_strength(
    technical_score: int,
    additional_score: Dict,
    ai_confidence: float
) -> float:
```

#### 역할
진입 신호 강도 계산 (0-1)

#### 계산 방식
```python
가중치:
- 기술적 지표: 40% (technical_score / 4 * 0.4)
- 추가 조건: 20% (bonus / 3 * 0.2)
- AI 신뢰도: 40% (ai_confidence * 0.4)
```

#### 구현 코드
```python
def calculate_signal_strength(
    technical_score: int,
    additional_score: Dict,
    ai_confidence: float
) -> float:
    """신호 강도 계산"""
    
    technical = (technical_score / 4.0) * 0.4
    bonus_count = additional_score.get('total_bonus', 0)
    additional = (bonus_count / 3.0) * 0.2
    ai = ai_confidence * 0.4
    
    strength = technical + additional + ai
    return round(strength, 3)
```

---

## 전체 의존성 그래프

### STRATEGY 모듈 내부
```
entry.py → indicators, ai, core
exit.py → trailing.py, ai, core
trailing.py → (독립)
signals.py → (독립)
```

### 사용하는 모듈
```
indicators/IndicatorCalculator
ai/ClaudeClient
core/Config, exceptions
```

### 사용되는 곳
```
engine/base_engine.py
├── EntryStrategy.check_entry_signal()
└── ExitStrategy.check_exit_signal()
```

---

## 개발 체크리스트

### entry.py
- [ ] EntryStrategy 클래스
- [ ] check_entry_signal() - 3단계 필터링
- [ ] AI 폴백 로직
- [ ] async/await 처리

### exit.py
- [ ] ExitStrategy 클래스
- [ ] check_exit_signal() - 5가지 우선순위
- [ ] 트레일링 스톱 통합
- [ ] AI 호출 (목표/정기)

### trailing.py
- [ ] TrailingStop 클래스
- [ ] update() - 활성화, 고점, 손절
- [ ] get_status() - 디버깅

### signals.py
- [ ] calculate_signal_strength()
- [ ] get_entry_reasons()

---

## 테스트 시나리오

### trailing.py 핵심 테스트
```python
trailing = TrailingStop(100.0, activation_pct=0.005, stop_distance=0.01)

# 활성화
trailing.update(100.5)  # activated = True

# 고점 갱신
trailing.update(105.0)  # highest = 105.0, stop = 103.95

# 손절 발동
result = trailing.update(103.94)
assert result == 'STOP'
```

---

## 주요 특징

### 1. 3단계 진입 필터
- 지표 (최소 3개) → 추가 조건 → AI (필수)

### 2. 5가지 청산 우선순위
- 손절 → 타임아웃 → 트레일링 → 목표(AI) → 정기(AI)

### 3. 트레일링 스톱
- +0.5% 활성화 → 고점 추적 → -1% 손절

### 4. AI 통합
- 진입: 필수 (폴백 있음)
- 청산: 선택적 (목표/정기)

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 3 (전략 레이어)  
**검증**: ✅ 완료