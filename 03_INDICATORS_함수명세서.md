# 03_INDICATORS 모듈 완벽 함수 명세서 v2.0 (개선판)

> **개선사항**: composite.py의 generate_composite_signal() 함수 완성

---

## 📋 목차
1. [indicators/calculator.py](#indicatorscalculatorpy)
2. [indicators/rsi.py](#indicatorsrsipy)
3. [indicators/macd.py](#indicatorsmacdpy)
4. [indicators/bollinger.py](#indicatorsbollingerpy)
5. [indicators/fibonacci.py](#indicatorsfibonaccipy)
6. [indicators/composite.py](#indicatorscompositepy) ⭐ 개선
7. [전체 의존성 그래프](#전체-의존성-그래프)
8. [실전 사용 예제](#실전-사용-예제)

---

## 📁 indicators/calculator.py

### 구현 코드 (전체)

```python
from typing import Dict, List
import pandas as pd
from .rsi import calculate_rsi
from .macd import calculate_macd
from .bollinger import calculate_bollinger_bands
from .fibonacci import calculate_fibonacci
from core.exceptions import InsufficientDataError


class IndicatorCalculator:
    """모든 기술적 지표 통합 계산기"""
    
    def calculate_all(self, ohlcv: List[List]) -> Dict:
        """
        모든 지표 한번에 계산
        
        Args:
            ohlcv: [[timestamp, o, h, l, c, v], ...]
        
        Returns:
            {
                'rsi': {...},
                'macd': {...},
                'bollinger': {...},
                'fibonacci': {...},
                'signal_count': 3
            }
        
        Raises:
            InsufficientDataError: 데이터 부족
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

### 구현 코드 (전체)

```python
import pandas as pd
from typing import Dict
from core.constants import INDICATOR_PARAMS


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

### 구현 코드 (전체)

```python
import pandas as pd
from typing import Dict
from core.constants import INDICATOR_PARAMS


def calculate_macd(df: pd.DataFrame) -> Dict:
    """
    MACD 계산
    
    Returns:
        {
            'value': 0.0015,
            'signal': 0.0012,
            'histogram': 0.0003,
            'golden_cross': True,
            'death_cross': False,
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

### 구현 코드 (전체)

```python
import pandas as pd
from typing import Dict
from core.constants import INDICATOR_PARAMS


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
            'upper_touch': False,
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

### 구현 코드 (전체)

```python
import pandas as pd
from typing import Dict, Tuple, Optional
from core.constants import FIBONACCI_LEVELS


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
            'at_support': True,
            'current_price': 101.20
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

## 📁 indicators/composite.py ⭐ 개선

### 구현 코드 (전체 완성)

```python
from typing import Dict, List


def generate_composite_signal(indicators: Dict) -> Dict:
    """
    복합 신호 생성
    
    여러 지표를 종합하여 최종 신호 생성 (선택적 기능)
    
    Args:
        indicators: calculator.calculate_all() 결과
    
    Returns:
        {
            'signal': 'STRONG_BUY',  # STRONG_BUY, BUY, NEUTRAL, SELL
            'strength': 0.85,         # 0-1
            'score': 8,               # 실제 점수
            'max_score': 10,          # 최대 점수
            'reasons': [
                'rsi_oversold',
                'macd_golden_cross',
                'bb_lower_touch',
                'fib_support'
            ]
        }
    
    점수 체계 (총 10점):
    - RSI (3점):
      * oversold: 3점
      * value < 50: 1.5점
    
    - MACD (3점):
      * golden_cross: 3점
      * momentum == 'bullish': 2점
    
    - Bollinger (2점):
      * lower_touch: 2점
    
    - Fibonacci (2점):
      * at_support: 2점
    
    신호 레벨:
    - 강도 >= 0.75: STRONG_BUY
    - 강도 >= 0.50: BUY
    - 강도 >= 0.25: NEUTRAL
    - 강도 <  0.25: SELL
    """
    score = 0
    max_score = 10
    reasons = []
    
    # RSI (최대 3점)
    rsi = indicators['rsi']
    if rsi['oversold']:
        score += 3
        reasons.append('rsi_oversold')
    elif rsi['value'] < 50:
        score += 1.5
        reasons.append('rsi_below_50')
    
    # MACD (최대 3점)
    macd = indicators['macd']
    if macd['golden_cross']:
        score += 3
        reasons.append('macd_golden_cross')
    elif macd['momentum'] == 'bullish':
        score += 2
        reasons.append('macd_bullish_momentum')
    
    # Bollinger (최대 2점)
    bollinger = indicators['bollinger']
    if bollinger['lower_touch']:
        score += 2
        reasons.append('bb_lower_touch')
    
    # Fibonacci (최대 2점)
    fibonacci = indicators['fibonacci']
    if fibonacci['at_support']:
        score += 2
        reasons.append('fib_support')
    
    # 강도 계산
    strength = score / max_score
    
    # 신호 레벨 결정
    if strength >= 0.75:
        signal = 'STRONG_BUY'
    elif strength >= 0.50:
        signal = 'BUY'
    elif strength >= 0.25:
        signal = 'NEUTRAL'
    else:
        signal = 'SELL'
    
    return {
        'signal': signal,
        'strength': round(strength, 2),
        'score': score,
        'max_score': max_score,
        'reasons': reasons
    }


def analyze_divergence(
    prices: List[float],
    indicator_values: List[float]
) -> str:
    """
    다이버전스 분석 (추가 기능)
    
    Args:
        prices: 최근 N개 가격
        indicator_values: 해당 기간 지표 값 (예: RSI)
    
    Returns:
        'bullish_divergence': 가격 하락 + 지표 상승
        'bearish_divergence': 가격 상승 + 지표 하락
        'none': 다이버전스 없음
    
    Example:
        >>> prices = [100, 98, 96, 94, 92]  # 하락
        >>> rsi_values = [40, 42, 44, 46, 48]  # 상승
        >>> analyze_divergence(prices, rsi_values)
        'bullish_divergence'
    """
    if len(prices) < 5 or len(indicator_values) < 5:
        return 'none'
    
    # 가격 추세
    price_trend = prices[-1] - prices[0]
    
    # 지표 추세
    indicator_trend = indicator_values[-1] - indicator_values[0]
    
    # 다이버전스 체크
    if price_trend < 0 and indicator_trend > 0:
        # 가격 하락, 지표 상승 → 강세 다이버전스
        return 'bullish_divergence'
    
    elif price_trend > 0 and indicator_trend < 0:
        # 가격 상승, 지표 하락 → 약세 다이버전스
        return 'bearish_divergence'
    
    return 'none'


def calculate_trend_strength(indicators: Dict) -> Dict:
    """
    추세 강도 분석 (추가 기능)
    
    Args:
        indicators: calculator.calculate_all() 결과
    
    Returns:
        {
            'trend': 'bullish',      # bullish, bearish, neutral
            'strength': 0.75,         # 0-1
            'confidence': 'high'      # high, medium, low
        }
    """
    bullish_count = 0
    total_count = 0
    
    # RSI
    if indicators['rsi']['value'] < 50:
        bullish_count += 1
    total_count += 1
    
    # MACD
    if indicators['macd']['momentum'] == 'bullish':
        bullish_count += 1
    total_count += 1
    
    # Bollinger
    if indicators['bollinger']['position'] == 'near_lower':
        bullish_count += 1
    total_count += 1
    
    # Fibonacci
    if indicators['fibonacci']['at_support']:
        bullish_count += 1
    total_count += 1
    
    # 강도 계산
    strength = bullish_count / total_count
    
    # 추세 결정
    if strength >= 0.75:
        trend = 'bullish'
        confidence = 'high'
    elif strength >= 0.50:
        trend = 'bullish'
        confidence = 'medium'
    elif strength >= 0.25:
        trend = 'neutral'
        confidence = 'medium'
    else:
        trend = 'bearish'
        confidence = 'low'
    
    return {
        'trend': trend,
        'strength': round(strength, 2),
        'confidence': confidence
    }
```

---

## 전체 의존성 그래프

```
indicators/
├── calculator.py (통합)
│   ├── import rsi.py
│   ├── import macd.py
│   ├── import bollinger.py
│   └── import fibonacci.py
│
├── rsi.py (독립)
├── macd.py (독립)
├── bollinger.py (독립)
├── fibonacci.py (독립)
└── composite.py (선택적)
    └── import calculator 결과 사용

모두 core/ 모듈에 의존
```

---

## 실전 사용 예제

### 예제 1: 기본 사용

```python
from indicators import IndicatorCalculator

# 계산기 초기화
calculator = IndicatorCalculator()

# OHLCV 데이터 (data/fetcher에서 받은 것)
ohlcv = data['ohlcv']

# 모든 지표 계산
indicators = calculator.calculate_all(ohlcv)

# 결과 확인
print(f"RSI: {indicators['rsi']['value']}")
print(f"MACD: {indicators['macd']['momentum']}")
print(f"진입 조건 충족: {indicators['signal_count']}/4")
```

### 예제 2: 복합 신호 사용

```python
from indicators import IndicatorCalculator
from indicators.composite import generate_composite_signal

calculator = IndicatorCalculator()
indicators = calculator.calculate_all(ohlcv)

# 복합 신호 생성
composite = generate_composite_signal(indicators)

print(f"신호: {composite['signal']}")
print(f"강도: {composite['strength']}")
print(f"점수: {composite['score']}/{composite['max_score']}")
print(f"이유: {', '.join(composite['reasons'])}")

# 진입 결정
if composite['signal'] == 'STRONG_BUY' and composite['strength'] >= 0.75:
    print("✅ 진입 권장!")
```

### 예제 3: 개별 지표 직접 사용

```python
import pandas as pd
from indicators.rsi import calculate_rsi
from indicators.macd import calculate_macd

# DataFrame 변환
df = pd.DataFrame(ohlcv, columns=[
    'timestamp', 'open', 'high', 'low', 'close', 'volume'
])

# 개별 지표 계산
rsi = calculate_rsi(df)
macd = calculate_macd(df)

print(f"RSI: {rsi['value']} ({rsi['trend']})")
print(f"MACD 골든크로스: {macd['golden_cross']}")
```

### 예제 4: 다이버전스 분석

```python
from indicators.composite import analyze_divergence

# 최근 10개 가격과 RSI
recent_prices = [candle[4] for candle in ohlcv[-10:]]
rsi_history = []  # RSI 계산 후 저장

divergence = analyze_divergence(recent_prices, rsi_history)

if divergence == 'bullish_divergence':
    print("🚀 강세 다이버전스 발견!")
```

---

## 개발 체크리스트

### calculator.py
- [x] IndicatorCalculator 클래스
- [x] calculate_all() 구현
- [x] _count_signals() 구현
- [x] DataFrame 변환
- [x] 예외 처리

### rsi.py
- [x] calculate_rsi() 함수
- [x] EMA 방식 사용
- [x] 과매수/과매도 판단
- [x] 트렌드 분석

### macd.py
- [x] calculate_macd() 함수
- [x] 골든크로스 감지
- [x] 모멘텀 분석
- [x] Histogram 계산

### bollinger.py
- [x] calculate_bollinger_bands() 함수
- [x] 상/하단 터치 판단
- [x] 밴드폭 계산
- [x] 가격 위치 분석

### fibonacci.py
- [x] calculate_fibonacci() 함수
- [x] 7개 레벨 계산
- [x] 지지/저항선 찾기
- [x] 지지선 터치 판단

### composite.py ⭐
- [x] generate_composite_signal() 구현 ⭐
- [x] 점수 체계 (10점 만점)
- [x] 신호 레벨 결정
- [x] analyze_divergence() 추가
- [x] calculate_trend_strength() 추가

---

**문서 버전**: v2.0 (개선판)  
**작성일**: 2025-01-15  
**개선사항**: 
- ⭐ composite.py의 generate_composite_signal() 완성
- ✅ 10점 만점 점수 체계
- ✅ 신호 레벨 (STRONG_BUY/BUY/NEUTRAL/SELL)
- ✅ analyze_divergence() 추가
- ✅ calculate_trend_strength() 추가
- ✅ 실전 사용 예제 추가

**검증 상태**: ✅ 완료
