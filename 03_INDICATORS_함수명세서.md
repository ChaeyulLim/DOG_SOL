# 03_INDICATORS 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [indicators/calculator.py](#indicatorscalculatorpy)
2. [indicators/rsi.py](#indicatorsrsipy)
3. [indicators/macd.py](#indicatorsmacdpy)
4. [indicators/bollinger.py](#indicatorsbollingerpy)
5. [indicators/fibonacci.py](#indicatorsfibonaccipy)
6. [indicators/composite.py](#indicatorscompositepy)
7. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📁 indicators/calculator.py

### 파일 전체 구조
```python
from typing import Dict, List
import pandas as pd
from .rsi import calculate_rsi
from .macd import calculate_macd
from .bollinger import calculate_bollinger_bands
from .fibonacci import calculate_fibonacci
from core.exceptions import InsufficientDataError

class IndicatorCalculator:
    def calculate_all(self, ohlcv: List[List]) -> Dict: ...
    def _count_signals(self, indicators: Dict) -> int: ...
```

---

### 📌 클래스: IndicatorCalculator

#### 목적
모든 기술적 지표를 한번에 계산하는 통합 인터페이스

---

### 📌 함수: IndicatorCalculator.calculate_all(ohlcv)

```python
def calculate_all(self, ohlcv: List[List]) -> Dict:
```

#### 역할
모든 기술적 지표(RSI, MACD, Bollinger, Fibonacci)를 한번에 계산

#### 인자
- `ohlcv: List[List]` - [[timestamp, open, high, low, close, volume], ...]

#### 사용하는 모듈/함수
1. `pandas.DataFrame()` - OHLCV를 DataFrame으로 변환
2. `calculate_rsi(df)` from indicators.rsi
3. `calculate_macd(df)` from indicators.macd
4. `calculate_bollinger_bands(df)` from indicators.bollinger
5. `calculate_fibonacci(df)` from indicators.fibonacci
6. `self._count_signals(indicators)` - 신호 개수 계산

#### 호출되는 곳
```python
# strategy/entry.py check_entry_conditions()
indicators = self.calculator.calculate_all(ohlcv)
if indicators['signal_count'] >= 3:
    # AI 호출

# engine/base_engine.py main_loop()
for symbol, data in all_data.items():
    indicators = self.calculator.calculate_all(data['ohlcv'])
```

#### 데이터 흐름
```
1. List[List] OHLCV 받음
2. pandas DataFrame 변환
3. 각 지표 계산 함수 호출
4. 결과 조합
5. 신호 개수 계산
6. 딕셔너리 반환
```

#### 반환값
```python
Dict:
    'rsi': Dict = {
        'value': 45.2,
        'oversold': False,
        'overbought': False,
        'trend': 'up'
    }
    'macd': Dict = {
        'value': 0.0015,
        'signal': 0.0012,
        'histogram': 0.0003,
        'golden_cross': True,
        'death_cross': False,
        'momentum': 'bullish'
    }
    'bollinger': Dict = {
        'upper': 0.3850,
        'middle': 0.3820,
        'lower': 0.3790,
        'position': 'near_lower',
        'lower_touch': True,
        'upper_touch': False,
        'bandwidth': 1.57
    }
    'fibonacci': Dict = {
        'levels': {
            '0.0': 0.3900,
            '0.236': 0.3876,
            ...
        },
        'support': ('0.618', 0.3818),
        'resistance': ('0.382', 0.3864),
        'at_support': True,
        'current_price': 0.3821
    }
    'signal_count': int = 3  # 충족된 조건 개수 (0-4)
```

#### 예외 처리
- `InsufficientDataError` - OHLCV < 50개

#### 구현 코드
```python
def calculate_all(self, ohlcv: List[List]) -> Dict:
    """
    모든 지표 한번에 계산
    
    Args:
        ohlcv: [[timestamp, o, h, l, c, v], ...]
    
    Returns:
        모든 지표 결과
    
    Raises:
        InsufficientDataError: 데이터 부족
    
    Example:
        >>> calc = IndicatorCalculator()
        >>> indicators = calc.calculate_all(ohlcv)
        >>> indicators['signal_count']
        3
    """
    if len(ohlcv) < 50:
        raise InsufficientDataError(
            f"최소 50개 캔들 필요 (현재: {len(ohlcv)})"
        )
    
    # DataFrame 변환
    df = pd.DataFrame(ohlcv, columns=[
        'timestamp', 'open', 'high', 'low', 'close', 'volume'
    ])
    
    # 각 지표 계산
    rsi = calculate_rsi(df)
    macd = calculate_macd(df)
    bollinger = calculate_bollinger_bands(df)
    fibonacci = calculate_fibonacci(df)
    
    # 결과 조합
    indicators = {
        'rsi': rsi,
        'macd': macd,
        'bollinger': bollinger,
        'fibonacci': fibonacci
    }
    
    # 신호 개수 계산
    indicators['signal_count'] = self._count_signals(indicators)
    
    return indicators
```

---

### 📌 함수: IndicatorCalculator._count_signals(indicators)

```python
def _count_signals(self, indicators: Dict) -> int:
```

#### 역할
진입 조건 충족 개수 계산 (내부 메서드)

#### 인자
- `indicators: Dict` - calculate_all()에서 계산된 지표들

#### 호출되는 곳
- `self.calculate_all()` 내부에서만

#### 진입 조건 (기획서 기준)
```
1. RSI < 70 (과매수 아님)
2. MACD 골든 크로스 OR 상승 모멘텀
3. 볼린저 하단 터치
4. 피보나치 지지선
```

#### 로직
```python
count = 0

# 조건 1: RSI < 70
if indicators['rsi']['value'] < 70:
    count += 1

# 조건 2: MACD 골든 크로스 or 상승
if indicators['macd']['golden_cross'] or \
   indicators['macd']['momentum'] == 'bullish':
    count += 1

# 조건 3: 볼린저 하단 터치
if indicators['bollinger']['lower_touch']:
    count += 1

# 조건 4: 피보나치 지지선
if indicators['fibonacci']['at_support']:
    count += 1

return count  # 0-4
```

#### 반환값
- `int`: 0-4 (충족된 조건 개수)

#### 구현 코드
```python
def _count_signals(self, indicators: Dict) -> int:
    """
    진입 조건 충족 개수
    
    조건:
    1. RSI < 70 (과매수 아님)
    2. MACD 골든 크로스 or 상승 모멘텀
    3. 볼린저 하단 터치
    4. 피보나치 지지선
    """
    count = 0
    
    # RSI
    if indicators['rsi']['value'] < 70:
        count += 1
    
    # MACD
    if indicators['macd']['golden_cross'] or \
       indicators['macd']['momentum'] == 'bullish':
        count += 1
    
    # Bollinger
    if indicators['bollinger']['lower_touch']:
        count += 1
    
    # Fibonacci
    if indicators['fibonacci']['at_support']:
        count += 1
    
    return count
```

---

## 📁 indicators/rsi.py

### 파일 전체 구조
```python
import pandas as pd
from typing import Dict
from core.constants import INDICATOR_PARAMS

def calculate_rsi(df: pd.DataFrame, period: int = None) -> Dict: ...
```

---

### 📌 함수: calculate_rsi(df, period)

```python
def calculate_rsi(df: pd.DataFrame, period: int = None) -> Dict:
```

#### 역할
RSI (Relative Strength Index) 계산

#### 인자
- `df: pd.DataFrame` - OHLCV 데이터프레임 (컬럼: timestamp, open, high, low, close, volume)
- `period: int = None` - 기간 (기본값: constants에서 가져옴, 14)

#### 사용하는 모듈/함수
- `constants.INDICATOR_PARAMS['RSI']` - 파라미터
- `pandas.Series.diff()` - 가격 변화
- `pandas.Series.ewm()` - 지수 이동 평균

#### 호출되는 곳
```python
# indicators/calculator.py calculate_all()
rsi = calculate_rsi(df)
```

#### RSI 계산 공식
```
1. delta = close.diff()
2. gain = delta.where(delta > 0, 0)
3. loss = -delta.where(delta < 0, 0)
4. avg_gain = gain.ewm(span=14).mean()
5. avg_loss = loss.ewm(span=14).mean()
6. RS = avg_gain / avg_loss
7. RSI = 100 - (100 / (1 + RS))
```

#### 반환값
```python
Dict:
    'value': float = 45.2           # RSI 값 (0-100)
    'oversold': bool = False        # < 30
    'overbought': bool = False      # > 70
    'trend': str = 'up'            # 'up', 'down', 'neutral'
```

#### 트렌드 판단 기준
```python
현재 RSI - 이전 RSI > +5: 'up'
현재 RSI - 이전 RSI < -5: 'down'
그 외: 'neutral'
```

#### 구현 코드
```python
def calculate_rsi(df: pd.DataFrame, period: int = None) -> Dict:
    """
    RSI 계산
    
    Args:
        df: OHLCV DataFrame
        period: 기간 (기본 14)
    
    Returns:
        {
            'value': 45.2,
            'oversold': False,
            'overbought': False,
            'trend': 'up'
        }
    
    Example:
        >>> rsi = calculate_rsi(df)
        >>> rsi['value']
        45.2
    """
    if period is None:
        period = INDICATOR_PARAMS['RSI']['period']
    
    # 가격 변화
    delta = df['close'].diff()
    
    # 상승/하락 분리
    gain = delta.where(delta > 0, 0)
    loss = -delta.where(delta < 0, 0)
    
    # 평균 계산 (EMA)
    avg_gain = gain.ewm(span=period, adjust=False).mean()
    avg_loss = loss.ewm(span=period, adjust=False).mean()
    
    # RS 계산
    rs = avg_gain / avg_loss
    
    # RSI 계산
    rsi = 100 - (100 / (1 + rs))
    
    current_rsi = float(rsi.iloc[-1])
    prev_rsi = float(rsi.iloc[-2])
    
    # 과매수/과매도
    oversold = current_rsi < INDICATOR_PARAMS['RSI']['oversold']
    overbought = current_rsi > INDICATOR_PARAMS['RSI']['overbought']
    
    # 트렌드
    if current_rsi > prev_rsi + 5:
        trend = 'up'
    elif current_rsi < prev_rsi - 5:
        trend = 'down'
    else:
        trend = 'neutral'
    
    return {
        'value': round(current_rsi, 2),
        'oversold': oversold,
        'overbought': overbought,
        'trend': trend
    }
```

---

## 📁 indicators/macd.py

### 파일 전체 구조
```python
import pandas as pd
from typing import Dict
from core.constants import INDICATOR_PARAMS

def calculate_macd(df: pd.DataFrame) -> Dict: ...
```

---

### 📌 함수: calculate_macd(df)

```python
def calculate_macd(df: pd.DataFrame) -> Dict:
```

#### 역할
MACD (Moving Average Convergence Divergence) 계산

#### 인자
- `df: pd.DataFrame` - OHLCV 데이터프레임

#### 사용하는 모듈/함수
- `constants.INDICATOR_PARAMS['MACD']` - {'fast': 12, 'slow': 26, 'signal': 9}
- `pandas.Series.ewm()` - 지수 이동 평균

#### 호출되는 곳
```python
# indicators/calculator.py calculate_all()
macd = calculate_macd(df)
```

#### MACD 계산 공식
```
1. EMA_fast = close.ewm(span=12).mean()
2. EMA_slow = close.ewm(span=26).mean()
3. MACD Line = EMA_fast - EMA_slow
4. Signal Line = MACD.ewm(span=9).mean()
5. Histogram = MACD - Signal
```

#### 골든 크로스 판단
```
이전 Histogram < 0 AND 현재 Histogram > 0
→ MACD가 Signal을 상향 돌파
```

#### 반환값
```python
Dict:
    'value': float = 0.0015           # MACD Line
    'signal': float = 0.0012          # Signal Line
    'histogram': float = 0.0003       # Histogram
    'golden_cross': bool = True       # 골든 크로스 발생
    'death_cross': bool = False       # 데드 크로스 발생
    'momentum': str = 'bullish'       # 'bullish', 'bearish', 'neutral'
```

#### 모멘텀 판단 기준
```python
현재 Histogram > 0 AND 증가 중: 'bullish'
현재 Histogram < 0 AND 감소 중: 'bearish'
그 외: 'neutral'
```

#### 구현 코드
```python
def calculate_macd(df: pd.DataFrame) -> Dict:
    """
    MACD 계산
    
    Args:
        df: OHLCV DataFrame
    
    Returns:
        {
            'value': 0.0015,
            'signal': 0.0012,
            'histogram': 0.0003,
            'golden_cross': True,
            'momentum': 'bullish'
        }
    """
    params = INDICATOR_PARAMS['MACD']
    
    # EMA 계산
    ema_fast = df['close'].ewm(span=params['fast'], adjust=False).mean()
    ema_slow = df['close'].ewm(span=params['slow'], adjust=False).mean()
    
    # MACD Line
    macd = ema_fast - ema_slow
    
    # Signal Line
    signal = macd.ewm(span=params['signal'], adjust=False).mean()
    
    # Histogram
    histogram = macd - signal
    
    # 현재/이전 값
    current_macd = float(macd.iloc[-1])
    current_signal = float(signal.iloc[-1])
    current_hist = float(histogram.iloc[-1])
    prev_hist = float(histogram.iloc[-2])
    
    # 골든 크로스 (MACD가 Signal 상향 돌파)
    golden_cross = (prev_hist < 0 and current_hist > 0)
    
    # 데드 크로스
    death_cross = (prev_hist > 0 and current_hist < 0)
    
    # 모멘텀
    if current_hist > 0 and current_hist > prev_hist:
        momentum = 'bullish'
    elif current_hist < 0 and current_hist < prev_hist:
        momentum = 'bearish'
    else:
        momentum = 'neutral'
    
    return {
        'value': round(current_macd, 6),
        'signal': round(current_signal, 6),
        'histogram': round(current_hist, 6),
        'golden_cross': golden_cross,
        'death_cross': death_cross,
        'momentum': momentum
    }
```

---

## 📁 indicators/bollinger.py

### 파일 전체 구조
```python
import pandas as pd
from typing import Dict
from core.constants import INDICATOR_PARAMS

def calculate_bollinger_bands(df: pd.DataFrame) -> Dict: ...
```

---

### 📌 함수: calculate_bollinger_bands(df)

```python
def calculate_bollinger_bands(df: pd.DataFrame) -> Dict:
```

#### 역할
볼린저밴드 계산

#### 인자
- `df: pd.DataFrame` - OHLCV 데이터프레임

#### 사용하는 모듈/함수
- `constants.INDICATOR_PARAMS['BOLLINGER']` - {'period': 20, 'std': 2.0}
- `pandas.Series.rolling()` - 이동 평균 및 표준편차

#### 호출되는 곳
```python
# indicators/calculator.py calculate_all()
bollinger = calculate_bollinger_bands(df)
```

#### 볼린저밴드 계산 공식
```
1. Middle Band = SMA(20) of close
2. Std = Standard Deviation(20) of close
3. Upper Band = Middle + (2 × Std)
4. Lower Band = Middle - (2 × Std)
5. Bandwidth = ((Upper - Lower) / Middle) × 100
```

#### 터치 판단 기준
```python
Lower Touch: 현재가 <= Lower × 1.005 (0.5% 여유)
Upper Touch: 현재가 >= Upper × 0.995 (0.5% 여유)
```

#### 반환값
```python
Dict:
    'upper': float = 0.3850           # 상단 밴드
    'middle': float = 0.3820          # 중간 밴드 (SMA)
    'lower': float = 0.3790           # 하단 밴드
    'position': str = 'near_lower'    # 가격 위치
    'lower_touch': bool = True        # 하단 터치
    'upper_touch': bool = False       # 상단 터치
    'bandwidth': float = 1.57         # 밴드폭 (%)
```

#### 포지션 값
```python
'near_upper': 상단 근처
'near_lower': 하단 근처
'middle': 중간
```

#### 구현 코드
```python
def calculate_bollinger_bands(df: pd.DataFrame) -> Dict:
    """
    볼린저밴드 계산
    
    Returns:
        {
            'upper': 0.3850,
            'middle': 0.3820,
            'lower': 0.3790,
            'position': 'near_lower',
            'lower_touch': True,
            'bandwidth': 1.57
        }
    """
    params = INDICATOR_PARAMS['BOLLINGER']
    period = params['period']
    std_multiplier = params['std']
    
    # Middle Band (SMA)
    middle = df['close'].rolling(window=period).mean()
    
    # Standard Deviation
    std = df['close'].rolling(window=period).std()
    
    # Upper/Lower Bands
    upper = middle + (std * std_multiplier)
    lower = middle - (std * std_multiplier)
    
    # 현재 값
    current_price = float(df['close'].iloc[-1])
    current_upper = float(upper.iloc[-1])
    current_middle = float(middle.iloc[-1])
    current_lower = float(lower.iloc[-1])
    
    # 밴드폭
    bandwidth = ((current_upper - current_lower) / current_middle) * 100
    
    # 가격 위치
    if current_price >= current_upper * 0.995:
        position = 'near_upper'
        upper_touch = True
        lower_touch = False
    elif current_price <= current_lower * 1.005:
        position = 'near_lower'
        upper_touch = False
        lower_touch = True
    else:
        position = 'middle'
        upper_touch = False
        lower_touch = False
    
    return {
        'upper': round(current_upper, 4),
        'middle': round(current_middle, 4),
        'lower': round(current_lower, 4),
        'position': position,
        'lower_touch': lower_touch,
        'upper_touch': upper_touch,
        'bandwidth': round(bandwidth, 2)
    }
```

---

## 📁 indicators/fibonacci.py

### 파일 전체 구조
```python
import pandas as pd
from typing import Dict, Tuple, Optional
from core.constants import FIBONACCI_LEVELS

def calculate_fibonacci(df: pd.DataFrame, period: int = 50) -> Dict: ...
```

---

### 📌 함수: calculate_fibonacci(df, period)

```python
def calculate_fibonacci(df: pd.DataFrame, period: int = 50) -> Dict:
```

#### 역할
피보나치 되돌림 레벨 계산

#### 인자
- `df: pd.DataFrame` - OHLCV 데이터프레임
- `period: int = 50` - 고점/저점 기준 기간

#### 사용하는 모듈/함수
- `constants.FIBONACCI_LEVELS` - [0.0, 0.236, 0.382, 0.5, 0.618, 0.786, 1.0]
- `pandas.Series.max()`, `pandas.Series.min()`

#### 호출되는 곳
```python
# indicators/calculator.py calculate_all()
fibonacci = calculate_fibonacci(df)
```

#### 피보나치 레벨 계산 공식
```
1. recent = df.tail(50)  # 최근 50개
2. high = recent['high'].max()
3. low = recent['low'].min()
4. diff = high - low

5. 각 레벨:
   level_0.0 = high - (diff × 0.0) = high
   level_0.236 = high - (diff × 0.236)
   level_0.382 = high - (diff × 0.382)
   level_0.5 = high - (diff × 0.5)
   level_0.618 = high - (diff × 0.618)
   level_0.786 = high - (diff × 0.786)
   level_1.0 = high - (diff × 1.0) = low
```

#### 지지선/저항선 찾기
```python
현재가보다 낮은 레벨 중 가장 가까운 = 지지선
현재가보다 높은 레벨 중 가장 가까운 = 저항선
```

#### 지지선 터치 판단
```python
abs(현재가 - 지지선) / 지지선 <= 0.005 (0.5% 이내)
```

#### 반환값
```python
Dict:
    'levels': Dict[str, float] = {
        '0.0': 0.3900,
        '0.236': 0.3876,
        '0.382': 0.3864,
        '0.5': 0.3850,
        '0.618': 0.3836,
        '0.786': 0.3818,
        '1.0': 0.3800
    }
    'support': Tuple[str, float] = ('0.618', 0.3818)     # 지지선
    'resistance': Tuple[str, float] = ('0.382', 0.3864)  # 저항선
    'at_support': bool = True                             # 지지선 근처
    'current_price': float = 0.3821
```

#### 구현 코드
```python
def calculate_fibonacci(df: pd.DataFrame, period: int = 50) -> Dict:
    """
    피보나치 되돌림 레벨 계산
    
    Args:
        df: OHLCV DataFrame
        period: 고점/저점 기준 기간
    
    Returns:
        {
            'levels': {'0.0': 100.0, '0.236': 102.36, ...},
            'support': ('0.618', 101.18),
            'resistance': ('0.382', 102.64),
            'at_support': True
        }
    """
    # 최근 N개 캔들에서 고점/저점
    recent = df.tail(period)
    high = float(recent['high'].max())
    low = float(recent['low'].min())
    
    # 현재가
    current_price = float(df['close'].iloc[-1])
    
    # 피보나치 레벨 계산
    diff = high - low
    levels = {}
    
    for level in FIBONACCI_LEVELS:
        price = high - (diff * level)
        levels[str(level)] = round(price, 4)
    
    # 가장 가까운 지지선/저항선 찾기
    support = None
    resistance = None
    
    for i, level in enumerate(FIBONACCI_LEVELS[1:], 1):
        level_price = levels[str(level)]
        
        if level_price < current_price:
            if support is None:
                support = (str(level), level_price)
        else:
            if resistance is None:
                resistance = (str(level), level_price)
                break
    
    # 지지선 근처 여부 (±0.5%)
    at_support = False
    if support:
        support_price = support[1]
        if abs(current_price - support_price) / support_price <= 0.005:
            at_support = True
    
    return {
        'levels': levels,
        'support': support,
        'resistance': resistance,
        'at_support': at_support,
        'current_price': round(current_price, 4)
    }
```

---

## 📁 indicators/composite.py

### 파일 전체 구조
```python
from typing import Dict

def generate_composite_signal(indicators: Dict) -> Dict: ...
```

---

### 📌 함수: generate_composite_signal(indicators)

```python
def generate_composite_signal(indicators: Dict) -> Dict:
```

#### 역할
복합 신호 생성 (여러 지표를 종합하여 최종 신호)

#### 인자
- `indicators: Dict` - calculator.calculate_all()의 반환값

#### 사용하는 모듈/함수
- 없음 (순수 로직)

#### 호출되는 곳
```python
# strategy/entry.py (선택적)
composite = generate_composite_signal(indicators)
if composite['signal'] == 'STRONG_BUY':
    # 추가 확신
```

#### 점수 계산 (총 10점)
```
RSI (3점):
  - oversold: 3점
  - value < 50: 1.5점

MACD (3점):
  - golden_cross: 3점
  - momentum == 'bullish': 2점

Bollinger (2점):
  - lower_touch: 2점

Fibonacci (2점):
  - at_support: 2점
```

#### 신호 레벨
```python
강도 >= 0.75 (7.5점 이상): 'STRONG_BUY'
강도 >= 0.50 (5.0점 이상): 'BUY'
강도 >= 0.25 (2.5점 이상): 'NEUTRAL'
강도 <  0.25: 'SELL'
```

#### 반환값
```python
Dict:
    'signal': str = 'STRONG_BUY'  # STRONG_BUY, BUY, NEUTRAL, SELL
    'strength': float = 0.85       # 0-1
    'score': int = 8               # 실제 점수
    'max_score': int = 10          # 최대 점수
    'reasons': List[str] = [
        'rsi_oversold',
        'macd_golden_cross',
        'bb_lower_touch',
        'fib_support'
    ]
```

#### 구현 코드
```python
def generate_composite_signal(indicators: Dict) -> Dict:
    """
    복합 신호 생성
    
    Args:
        indicators: calculator.calculate_all() 결과
    
    Returns:
        {
            'signal': 'STRONG_BUY',
            'strength': 0.85,
            'reasons': ['rsi_oversold', 'macd_golden']
        }
    """
    score = 0
    max_