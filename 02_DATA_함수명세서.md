# 02_DATA 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [data/fetcher.py](#datafetcherpy)
2. [data/cache.py](#datacachepy)
3. [data/processor.py](#dataprocessorpy)
4. [data/historical.py](#datahistoricalpy)
5. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📁 data/fetcher.py

### 파일 전체 구조
```python
import time
import ccxt
from typing import Dict, List, Optional
from core.constants import SYMBOLS, CANDLE_LIMIT, INDICATOR_TIMEFRAME
from core.exceptions import DataFetchError, NetworkError
from .cache import DataCache
from .processor import DataProcessor

class MarketDataFetcher:
    def __init__(self, exchange: ccxt.Exchange): ...
    
    async def fetch_market_data(self, symbol: str) -> Dict: ...
    async def _fetch_ticker(self, symbol: str) -> Dict: ...
    async def _fetch_ohlcv(self, symbol: str) -> List[List]: ...
    async def _fetch_orderbook(self, symbol: str, limit: int = 20) -> Dict: ...
    async def fetch_multiple_symbols(self) -> Dict[str, Dict]: ...
```

---

### 📌 클래스: MarketDataFetcher

#### 목적
실시간 시장 데이터 수집 및 캐싱

---

### 📌 함수: MarketDataFetcher.__init__(exchange)

```python
def __init__(self, exchange: ccxt.Exchange):
```

#### 역할
데이터 수집기 초기화 (Exchange 인스턴스 받음)

#### 인자
- `exchange: ccxt.Exchange` - 이미 초기화된 ccxt.bybit() 인스턴스

#### 사용하는 모듈/함수
- `DataCache()` - 캐시 시스템 초기화
- `DataProcessor()` - 데이터 정제기 초기화

#### 호출되는 곳
```python
# engine/base_engine.py __init__()
from data import MarketDataFetcher

self.fetcher = MarketDataFetcher(self.exchange)
```

#### 초기화 내용
```python
self.exchange = exchange          # ccxt.Exchange 인스턴스
self.cache = DataCache()          # 캐시 시스템
self.processor = DataProcessor()   # 데이터 정제기
```

#### 구현 코드
```python
def __init__(self, exchange: ccxt.Exchange):
    """
    초기화
    
    Args:
        exchange: ccxt Exchange 인스턴스
    """
    self.exchange = exchange
    self.cache = DataCache()
    self.processor = DataProcessor()
```

---

### 📌 함수: MarketDataFetcher.fetch_market_data(symbol)

```python
async def fetch_market_data(self, symbol: str) -> Dict:
```

#### 역할
단일 심볼의 실시간 시장 데이터 수집 (캐싱 적용)

#### 인자
- `symbol: str` - 'DOGE/USDT' 또는 'SOL/USDT'

#### 사용하는 모듈/함수
1. `self.cache.get(symbol)` - 캐시 확인 (먼저)
2. `self._fetch_ticker(symbol)` - 현재가 조회
3. `self._fetch_ohlcv(symbol)` - OHLCV 캔들 조회
4. `self._fetch_orderbook(symbol)` - 호가창 조회
5. `self.processor.validate_and_clean(market_data)` - 데이터 검증
6. `self.cache.set(symbol, market_data)` - 캐싱 저장

#### 호출되는 곳
```python
# engine/base_engine.py main_loop()
data = await self.fetcher.fetch_market_data('DOGE/USDT')

# self.fetch_multiple_symbols() 내부
for symbol in SYMBOLS:
    data = await self.fetch_market_data(symbol)
```

#### 데이터 흐름
```
1. 캐시 확인 (있으면 즉시 반환)
2. Bybit API 호출 (ticker, ohlcv, orderbook)
3. 데이터 조합
4. 검증 및 정제
5. 캐싱 후 반환
```

#### 반환값
```python
Dict:
    'symbol': str = 'DOGE/USDT'
    'price': float = 0.3821
    'volume_24h': float = 1234567890.0
    'change_24h': float = 2.5  # 퍼센트
    'ohlcv': List[List] = [
        [timestamp, open, high, low, close, volume],
        ...  # 200개
    ]
    'orderbook': Dict = {
        'bids': [[price, amount], ...],  # 매수 호가 20개
        'asks': [[price, amount], ...]   # 매도 호가 20개
    }
    'timestamp': int = 1234567890
```

#### 예외 처리
- `DataFetchError` - 데이터 수집 실패 시

#### 구현 코드
```python
async def fetch_market_data(self, symbol: str) -> Dict:
    """
    시장 데이터 수집 (캐싱 적용)
    
    Called by:
        - engine/base_engine.py: main_loop()
    
    Args:
        symbol: 'DOGE/USDT'
    
    Returns:
        시장 데이터 딕셔너리
    
    Raises:
        DataFetchError: 데이터 수집 실패
    
    Example:
        >>> fetcher = MarketDataFetcher(exchange)
        >>> data = await fetcher.fetch_market_data('DOGE/USDT')
        >>> data['price']
        0.3821
    """
    # 캐시 확인
    cached = self.cache.get(symbol)
    if cached:
        return cached
    
    try:
        # 현재가 & 24h 거래량
        ticker = await self._fetch_ticker(symbol)
        
        # OHLCV 캔들
        ohlcv = await self._fetch_ohlcv(symbol)
        
        # 호가창
        orderbook = await self._fetch_orderbook(symbol)
        
        # 데이터 조합
        market_data = {
            'symbol': symbol,
            'price': ticker['last'],
            'volume_24h': ticker['quoteVolume'],
            'change_24h': ticker['percentage'],
            'ohlcv': ohlcv,
            'orderbook': orderbook,
            'timestamp': int(time.time())
        }
        
        # 데이터 검증 및 정제
        market_data = self.processor.validate_and_clean(market_data)
        
        # 캐시 저장
        self.cache.set(symbol, market_data)
        
        return market_data
    
    except Exception as e:
        raise DataFetchError(f"데이터 수집 실패 ({symbol}): {e}")
```

---

### 📌 함수: MarketDataFetcher._fetch_ticker(symbol)

```python
async def _fetch_ticker(self, symbol: str) -> Dict:
```

#### 역할
현재가 및 24시간 거래량 조회 (내부 메서드)

#### 인자
- `symbol: str`

#### 사용하는 모듈/함수
- `self.exchange.fetch_ticker(symbol)` - ccxt API

#### 호출되는 곳
- `self.fetch_market_data()` 내부에서만

#### 반환값
```python
Dict:
    'last': float        # 현재가
    'quoteVolume': float # 24h 거래량 (USDT 기준)
    'percentage': float  # 24h 변화율
```

#### 예외 처리
```python
- ccxt.NetworkError → NetworkError로 변환
- 기타 Exception → DataFetchError로 변환
- percentage가 None이면 0으로 처리
```

#### 구현 코드
```python
async def _fetch_ticker(self, symbol: str) -> Dict:
    """
    현재가 정보 수집
    
    Returns:
        {
            'last': 0.3821,
            'quoteVolume': 1234567890,
            'percentage': 2.5
        }
    """
    try:
        ticker = self.exchange.fetch_ticker(symbol)
        return {
            'last': ticker['last'],
            'quoteVolume': ticker['quoteVolume'],
            'percentage': ticker['percentage'] or 0
        }
    except ccxt.NetworkError as e:
        raise NetworkError(f"네트워크 오류: {e}")
    except Exception as e:
        raise DataFetchError(f"Ticker 수집 실패: {e}")
```

---

### 📌 함수: MarketDataFetcher._fetch_ohlcv(symbol)

```python
async def _fetch_ohlcv(self, symbol: str) -> List[List]:
```

#### 역할
OHLCV 캔들 데이터 조회 (내부 메서드)

#### 인자
- `symbol: str`

#### 사용하는 모듈/함수
- `self.exchange.fetch_ohlcv(symbol, timeframe, limit)`
- `constants.INDICATOR_TIMEFRAME` - '5m'
- `constants.CANDLE_LIMIT` - 200

#### 호출되는 곳
- `self.fetch_market_data()` 내부에서만

#### 반환값
```python
List[List]:
    [
        [timestamp, open, high, low, close, volume],
        [timestamp, open, high, low, close, volume],
        ...  # 200개
    ]
```

#### 구현 코드
```python
async def _fetch_ohlcv(self, symbol: str) -> List[List]:
    """
    OHLCV 캔들 데이터 수집
    
    Returns:
        [[timestamp, open, high, low, close, volume], ...]
    """
    try:
        ohlcv = self.exchange.fetch_ohlcv(
            symbol,
            timeframe=INDICATOR_TIMEFRAME,
            limit=CANDLE_LIMIT
        )
        return ohlcv
    except Exception as e:
        raise DataFetchError(f"OHLCV 수집 실패: {e}")
```

---

### 📌 함수: MarketDataFetcher._fetch_orderbook(symbol, limit)

```python
async def _fetch_orderbook(self, symbol: str, limit: int = 20) -> Dict:
```

#### 역할
호가창 데이터 조회 (내부 메서드, 선택적)

#### 인자
- `symbol: str`
- `limit: int = 20` - 상위 N개 호가

#### 사용하는 모듈/함수
- `self.exchange.fetch_order_book(symbol, limit)`

#### 호출되는 곳
- `self.fetch_market_data()` 내부에서만

#### 반환값
```python
Dict:
    'bids': List[List] = [[price, amount], ...]  # 매수 호가
    'asks': List[List] = [[price, amount], ...]  # 매도 호가
    
# 실패 시 (필수 아님)
    'bids': []
    'asks': []
```

#### 구현 코드
```python
async def _fetch_orderbook(self, symbol: str, limit: int = 20) -> Dict:
    """
    호가창 데이터 수집
    
    Args:
        symbol: 심볼
        limit: 호가 개수 (기본 20)
    
    Returns:
        {
            'bids': [[price, amount], ...],
            'asks': [[price, amount], ...]
        }
    """
    try:
        orderbook = self.exchange.fetch_order_book(symbol, limit=limit)
        return {
            'bids': orderbook['bids'][:limit],
            'asks': orderbook['asks'][:limit]
        }
    except Exception as e:
        # 호가창은 필수 아님, 빈 값 반환
        return {'bids': [], 'asks': []}
```

---

### 📌 함수: MarketDataFetcher.fetch_multiple_symbols()

```python
async def fetch_multiple_symbols(self) -> Dict[str, Dict]:
```

#### 역할
모든 심볼 데이터 동시 수집

#### 인자
- 없음 (self만)

#### 사용하는 모듈/함수
- `constants.SYMBOLS` - ['DOGE/USDT', 'SOL/USDT']
- `self.fetch_market_data(symbol)` 반복 호출

#### 호출되는 곳
```python
# engine/base_engine.py main_loop()
all_data = await self.fetcher.fetch_multiple_symbols()

for symbol, data in all_data.items():
    # 각 심볼 처리
    indicators = self.calculator.calculate_all(data['ohlcv'])
```

#### 반환값
```python
Dict[str, Dict]:
    {
        'DOGE/USDT': {
            'symbol': 'DOGE/USDT',
            'price': 0.3821,
            'ohlcv': [...],
            ...
        },
        'SOL/USDT': {
            'symbol': 'SOL/USDT',
            'price': 145.32,
            'ohlcv': [...],
            ...
        }
    }
```

#### 특이사항
- 하나 실패해도 계속 진행
- 실패한 심볼은 딕셔너리에서 누락
- 로그에 경고 출력

#### 구현 코드
```python
async def fetch_multiple_symbols(self) -> Dict[str, Dict]:
    """
    모든 심볼 데이터 동시 수집
    
    Called by:
        - engine/base_engine.py (메인 루프)
    
    Returns:
        {
            'DOGE/USDT': {...},
            'SOL/USDT': {...}
        }
    
    Example:
        >>> data = await fetcher.fetch_multiple_symbols()
        >>> data['DOGE/USDT']['price']
        0.3821
    """
    result = {}
    
    for symbol in SYMBOLS:
        try:
            data = await self.fetch_market_data(symbol)
            result[symbol] = data
        except DataFetchError as e:
            # 하나 실패해도 계속 진행
            print(f"⚠️ {symbol} 수집 실패: {e}")
            continue
    
    return result
```

---

## 📁 data/cache.py

### 파일 전체 구조
```python
import time
from typing import Dict, Optional
from core.constants import CACHE_TTL

class DataCache:
    def __init__(self, ttl: int = CACHE_TTL): ...
    def get(self, key: str) -> Optional[Dict]: ...
    def set(self, key: str, data: Dict) -> None: ...
    def clear(self, key: Optional[str] = None) -> None: ...
    def get_stats(self) -> Dict: ...
```

---

### 📌 클래스: DataCache

#### 목적
메모리 기반 데이터 캐시 (1분간 유효)

---

### 📌 함수: DataCache.__init__(ttl)

```python
def __init__(self, ttl: int = CACHE_TTL):
```

#### 역할
캐시 시스템 초기화

#### 인자
- `ttl: int = 60` - 캐시 유효 시간 (초)

#### 사용하는 모듈/함수
- `constants.CACHE_TTL` - 60

#### 호출되는 곳
```python
# data/fetcher.py __init__()
self.cache = DataCache()
```

#### 초기화 내용
```python
self.ttl = ttl                    # 유효 시간 (60초)
self._cache: Dict[str, Dict] = {} # 빈 딕셔너리
```

#### 구현 코드
```python
def __init__(self, ttl: int = CACHE_TTL):
    """
    초기화
    
    Args:
        ttl: 캐시 유효 시간 (초)
    """
    self.ttl = ttl
    self._cache: Dict[str, Dict] = {}
```

---

### 📌 함수: DataCache.get(key)

```python
def get(self, key: str) -> Optional[Dict]:
```

#### 역할
캐시 조회 (만료 확인 포함)

#### 인자
- `key: str` - 심볼 ('DOGE/USDT')

#### 사용하는 모듈/함수
- `time.time()` - 현재 시간
- `self._cache` - 내부 딕셔너리

#### 호출되는 곳
```python
# data/fetcher.py fetch_market_data() 시작 부분
cached = self.cache.get(symbol)
if cached:
    return cached  # 캐시 히트
```

#### 로직
```python
1. key가 _cache에 없으면 → None 반환
2. key가 있으면:
   - 현재시간 - cached_at > ttl?
     - Yes: 만료, 삭제 후 None 반환
     - No: 유효, data 반환
```

#### 반환값
- `Optional[Dict]`:
  - 캐시 히트: market_data 딕셔너리
  - 캐시 미스/만료: None

#### 구현 코드
```python
def get(self, key: str) -> Optional[Dict]:
    """
    캐시 조회
    
    Args:
        key: 심볼 ('DOGE/USDT')
    
    Returns:
        캐시된 데이터 또는 None
    
    Example:
        >>> cache = DataCache()
        >>> data = cache.get('DOGE/USDT')
    """
    if key not in self._cache:
        return None
    
    entry = self._cache[key]
    
    # 만료 확인
    if time.time() - entry['cached_at'] > self.ttl:
        del self._cache[key]
        return None
    
    return entry['data']
```

---

### 📌 함수: DataCache.set(key, data)

```python
def set(self, key: str, data: Dict) -> None:
```

#### 역할
캐시 저장

#### 인자
- `key: str` - 심볼
- `data: Dict` - market_data

#### 사용하는 모듈/함수
- `time.time()` - 현재 시간

#### 호출되는 곳
```python
# data/fetcher.py fetch_market_data() 끝 부분
self.cache.set(symbol, market_data)
```

#### 저장 형식
```python
self._cache[key] = {
    'data': data,
    'cached_at': time.time()
}
```

#### 구현 코드
```python
def set(self, key: str, data: Dict) -> None:
    """
    캐시 저장
    
    Args:
        key: 심볼
        data: 시장 데이터
    """
    self._cache[key] = {
        'data': data,
        'cached_at': time.time()
    }
```

---

### 📌 함수: DataCache.clear(key)

```python
def clear(self, key: Optional[str] = None) -> None:
```

#### 역할
캐시 삭제

#### 인자
- `key: Optional[str] = None`
  - None: 전체 삭제
  - 특정 키: 해당 키만 삭제

#### 호출되는 곳
```python
# 테스트 시
cache.clear()  # 전체 삭제

# 특정 심볼 강제 갱신 시
cache.clear('DOGE/USDT')
```

#### 구현 코드
```python
def clear(self, key: Optional[str] = None) -> None:
    """
    캐시 삭제
    
    Args:
        key: 특정 키 (None이면 전체 삭제)
    """
    if key:
        if key in self._cache:
            del self._cache[key]
    else:
        self._cache.clear()
```

---

### 📌 함수: DataCache.get_stats()

```python
def get_stats(self) -> Dict:
```

#### 역할
캐시 통계 조회 (디버깅/모니터링용)

#### 반환값
```python
Dict:
    'total_keys': int      # 총 키 개수
    'valid_keys': int      # 유효한 키 개수
    'expired_keys': int    # 만료된 키 개수
```

#### 구현 코드
```python
def get_stats(self) -> Dict:
    """
    캐시 통계
    
    Returns:
        {
            'total_keys': 2,
            'valid_keys': 1,
            'expired_keys': 1
        }
    """
    total = len(self._cache)
    valid = 0
    
    current_time = time.time()
    for entry in self._cache.values():
        if current_time - entry['cached_at'] <= self.ttl:
            valid += 1
    
    return {
        'total_keys': total,
        'valid_keys': valid,
        'expired_keys': total - valid
    }
```

---

## 📁 data/processor.py

### 파일 전체 구조
```python
from typing import Dict, List
import numpy as np
from core.exceptions import DataValidationError, InsufficientDataError

class DataProcessor:
    def validate_and_clean(self, market_data: Dict) -> Dict: ...
    def _clean_ohlcv(self, ohlcv: List[List]) -> List[List]: ...
    def detect_price_spike(
        self,
        current_price: float,
        recent_prices: List[float],
        threshold: float = 3.0
    ) -> bool: ...
    def normalize_price(
        self,
        prices: List[float],
        method: str = 'minmax'
    ) -> List[float]: ...
```

---

### 📌 클래스: DataProcessor

#### 목적
데이터 검증, 정제, 이상치 감지
    
    Example:
        >>> processor = DataProcessor()
        >>> is_spike = processor.detect_price_spike(
        ...     100.0,
        ...     [90, 91, 92, 93, 94]
        ... )
        >>> is_spike
        False
    """
    if len(recent_prices) < 20:
        return False
    
    mean = np.mean(recent_prices)
    std = np.std(recent_prices)
    
    upper = mean + (threshold * std)
    lower = mean - (threshold * std)
    
    if current_price > upper or current_price < lower:
        deviation = abs(current_price - mean) / mean * 100
        return True
    
    return False
```

---

### 📌 함수: DataProcessor.normalize_price(prices, method)

```python
def normalize_price(
    self,
    prices: List[float],
    method: str = 'minmax'
) -> List[float]:
```

#### 역할
가격 정규화 (선택적 기능)

#### 인자
- `prices: List[float]` - 가격 리스트
- `method: str = 'minmax'` - 'minmax' 또는 'zscore'

#### 사용하는 모듈/함수
- `numpy.array()`
- `numpy.min()`, `numpy.max()`
- `numpy.mean()`, `numpy.std()`

#### 호출되는 곳
```python
# indicators/calculator.py (선택적)
normalized = processor.normalize_price(close_prices)
```

#### 정규화 방법
**MinMax (0~1)**:
```python
(x - min) / (max - min)
```

**Z-Score**:
```python
(x - mean) / std
```

#### 반환값
- `List[float]`: 정규화된 가격

#### 구현 코드
```python
def normalize_price(
    self,
    prices: List[float],
    method: str = 'minmax'
) -> List[float]:
    """
    가격 정규화
    
    Args:
        prices: 가격 리스트
        method: 'minmax' or 'zscore'
    
    Returns:
        정규화된 가격
    """
    prices_array = np.array(prices)
    
    if method == 'minmax':
        min_val = prices_array.min()
        max_val = prices_array.max()
        return ((prices_array - min_val) / (max_val - min_val)).tolist()
    
    elif method == 'zscore':
        mean = prices_array.mean()
        std = prices_array.std()
        return ((prices_array - mean) / std).tolist()
    
    return prices
```

---

## 📁 data/historical.py

### 파일 전체 구조
```python
import pandas as pd
from typing import Dict, List
from datetime import datetime
from pathlib import Path
from core.constants import SYMBOLS, symbol_to_filename
from core.exceptions import DataFetchError
from .processor import DataProcessor

class HistoricalDataLoader:
    def __init__(self, data_dir: str = 'storage/historical'): ...
    
    def load_historical_data(
        self,
        symbol: str,
        start_date: str,
        end_date: str
    ) -> List[Dict]: ...
    
    def _clean_historical_data(self, data: List[Dict]) -> List[Dict]: ...
    
    def download_historical_data(
        self,
        symbol: str,
        start_date: str,
        timeframe: str = '5m'
    ) -> None: ...
```

---

### 📌 클래스: HistoricalDataLoader

#### 목적
백테스트용 과거 데이터 로드 (CSV 파일)

---

### 📌 함수: HistoricalDataLoader.__init__(data_dir)

```python
def __init__(self, data_dir: str = 'storage/historical'):
```

#### 역할
과거 데이터 로더 초기화

#### 인자
- `data_dir: str = 'storage/historical'` - 데이터 디렉토리 경로

#### 사용하는 모듈/함수
- `pathlib.Path()`
- `DataProcessor()` - 데이터 정제기

#### 호출되는 곳
```python
# engine/backtest_engine.py __init__()
from data import HistoricalDataLoader

self.loader = HistoricalDataLoader()
```

#### 초기화 내용
```python
self.data_dir = Path(data_dir)    # Path 객체
self.processor = DataProcessor()   # 정제기
```

#### 구현 코드
```python
def __init__(self, data_dir: str = 'storage/historical'):
    """
    초기화
    
    Args:
        data_dir: 데이터 디렉토리
    """
    self.data_dir = Path(data_dir)
    self.processor = DataProcessor()
```

---

### 📌 함수: HistoricalDataLoader.load_historical_data(symbol, start_date, end_date)

```python
def load_historical_data(
    self,
    symbol: str,
    start_date: str,
    end_date: str
) -> List[Dict]:
```

#### 역할
CSV 파일에서 과거 데이터 로드

#### 인자
- `symbol: str` - 'DOGE/USDT'
- `start_date: str` - '2024-01-01'
- `end_date: str` - '2024-12-31'

#### 사용하는 모듈/함수
- `constants.symbol_to_filename(symbol)` - 파일명 변환
- `pandas.read_csv(filepath)`
- `pandas.to_datetime()`
- `self._clean_historical_data(data)`

#### 호출되는 곳
```python
# engine/backtest_engine.py __init__()
for symbol in SYMBOLS:
    data = self.loader.load_historical_data(
        symbol,
        '2024-01-01',
        '2024-12-31'
    )
    self.historical_data[symbol] = data
```

#### 파일 경로
```python
symbol_to_filename('DOGE/USDT') = 'DOGE_USDT'
filepath = 'storage/historical/DOGE_USDT.csv'
```

#### CSV 형식
```csv
timestamp,open,high,low,close,volume
1704067200,0.3821,0.3850,0.3800,0.3835,1000000
1704067500,0.3835,0.3860,0.3820,0.3840,950000
...
```

#### 반환값
```python
List[Dict]:
    [
        {
            'timestamp': 1704067200,
            'open': 0.3821,
            'high': 0.3850,
            'low': 0.3800,
            'close': 0.3835,
            'volume': 1000000.0
        },
        ...
    ]
```

#### 예외 처리
- `DataFetchError` - 파일 없음 또는 로드 실패

#### 구현 코드
```python
def load_historical_data(
    self,
    symbol: str,
    start_date: str,
    end_date: str
) -> List[Dict]:
    """
    과거 데이터 로드
    
    Args:
        symbol: 'DOGE/USDT'
        start_date: '2024-01-01'
        end_date: '2024-12-31'
    
    Returns:
        [
            {
                'timestamp': 1234567890,
                'open': 100.0,
                'high': 102.0,
                'low': 99.0,
                'close': 101.0,
                'volume': 1000000
            },
            ...
        ]
    
    Raises:
        DataFetchError: 파일 없음 또는 로드 실패
    """
    filename = symbol_to_filename(symbol)
    filepath = self.data_dir / f"{filename}.csv"
    
    if not filepath.exists():
        raise DataFetchError(
            f"데이터 파일 없음: {filepath}\n"
            f"Bybit에서 다운로드 필요"
        )
    
    try:
        # CSV 로드
        df = pd.read_csv(filepath)
        
        # 날짜 필터링
        df['date'] = pd.to_datetime(df['timestamp'], unit='s')
        start = pd.to_datetime(start_date)
        end = pd.to_datetime(end_date)
        
        df = df[(df['date'] >= start) & (df['date'] <= end)]
        
        # 딕셔너리 변환
        data = df.to_dict('records')
        
        # 정제
        data = self._clean_historical_data(data)
        
        return data
    
    except Exception as e:
        raise DataFetchError(f"CSV 로드 실패: {e}")
```

---

### 📌 함수: HistoricalDataLoader._clean_historical_data(data)

```python
def _clean_historical_data(self, data: List[Dict]) -> List[Dict]:
```

#### 역할
과거 데이터 정제 (내부 메서드)

#### 인자
- `data: List[Dict]` - CSV에서 로드한 원본 데이터

#### 호출되는 곳
- `self.load_historical_data()` 내부에서만

#### 정제 작업
1. 필수 필드 확인
2. 가격 유효성 검증
3. 이상치 제거

#### 반환값
- `List[Dict]`: 정제된 데이터

#### 구현 코드
```python
def _clean_historical_data(self, data: List[Dict]) -> List[Dict]:
    """과거 데이터 정제"""
    cleaned = []
    
    for candle in data:
        # 필수 필드 확인
        required = ['timestamp', 'open', 'high', 'low', 'close', 'volume']
        if not all(k in candle for k in required):
            continue
        
        # 가격 유효성
        if any(candle[k] <= 0 for k in ['open', 'high', 'low', 'close']):
            continue
        
        cleaned.append(candle)
    
    return cleaned
```

---

### 📌 함수: HistoricalDataLoader.download_historical_data(symbol, start_date, timeframe)

```python
def download_historical_data(
    self,
    symbol: str,
    start_date: str,
    timeframe: str = '5m'
) -> None:
```

#### 역할
Bybit에서 과거 데이터 다운로드 (스켈레톤)

#### 인자
- `symbol: str` - 'DOGE/USDT'
- `start_date: str` - '2024-01-01'
- `timeframe: str = '5m'` - 타임프레임

#### 구현 상태
- 현재: 스켈레톤만 제공
- 실제 구현: Bybit API 사용하여 반복 호출 필요

#### 구현 가이드
```python
def download_historical_data(
    self,
    symbol: str,
    start_date: str,
    timeframe: str = '5m'
) -> None:
    """
    Bybit에서 과거 데이터 다운로드
    
    Args:
        symbol: 'DOGE/USDT'
        start_date: '2024-01-01'
        timeframe: '5m'
    
    Note:
        실제 구현은 Bybit API 사용
        여기서는 스켈레톤만 제공
    """
    # TODO: 실제 구현
    # 1. start_date부터 현재까지 기간 계산
    # 2. 1000개씩 반복 호출 (Bybit 제한)
    # 3. DataFrame으로 조합
    # 4. CSV로 저장
    pass
```

---

## 전체 의존성 그래프

### DATA 모듈 내부 의존성
```
fetcher.py
├── import cache.py (DataCache)
├── import processor.py (DataProcessor)
└── import core

cache.py
└── import core

processor.py
└── import core

historical.py
├── import processor.py (DataProcessor)
└── import core
```

### DATA 모듈이 사용하는 것
```
core/
├── constants.py (SYMBOLS, CANDLE_LIMIT, CACHE_TTL 등)
└── exceptions.py (DataFetchError, NetworkError 등)

외부 라이브러리:
├── ccxt (Exchange API)
├── pandas (CSV 처리)
└── numpy (수치 계산)
```

### DATA 모듈을 사용하는 곳
```
engine/base_engine.py
├── MarketDataFetcher.fetch_multiple_symbols()
└── indicators로 데이터 전달

engine/backtest_engine.py
└── HistoricalDataLoader.load_historical_data()
```

---

## 전체 호출 흐름

### 실시간 데이터 수집 흐름
```
1. engine/base_engine.py main_loop()
   ↓
2. fetcher.fetch_multiple_symbols()
   ↓
3. for symbol in SYMBOLS:
   ↓
4. fetcher.fetch_market_data(symbol)
   ↓
5. cache.get(symbol) → 히트면 반환
   ↓
6. _fetch_ticker(symbol)
7. _fetch_ohlcv(symbol)
8. _fetch_orderbook(symbol)
   ↓
9. processor.validate_and_clean(data)
   ↓
10. cache.set(symbol, data)
    ↓
11. return data
    ↓
12. indicators/calculator.py로 전달
```

### 백테스트 데이터 로드 흐름
```
1. engine/backtest_engine.py __init__()
   ↓
2. loader = HistoricalDataLoader()
   ↓
3. for symbol in SYMBOLS:
   ↓
4. data = loader.load_historical_data(symbol, start, end)
   ↓
5. CSV 파일 읽기
   ↓
6. 날짜 필터링
   ↓
7. _clean_historical_data()
   ↓
8. return List[Dict]
   ↓
9. backtest_engine에서 재생
```

---

## 개발 체크리스트

### fetcher.py
- [ ] MarketDataFetcher 클래스 정의
- [ ] __init__() - exchange, cache, processor 초기화
- [ ] fetch_market_data() - 캐싱 로직 포함
- [ ] _fetch_ticker() - ccxt API 호출
- [ ] _fetch_ohlcv() - 200개 캔들 조회
- [ ] _fetch_orderbook() - 실패해도 계속 진행
- [ ] fetch_multiple_symbols() - 모든 심볼 순회
- [ ] async/await 비동기 처리
- [ ] 예외를 DataFetchError로 래핑

### cache.py
- [ ] DataCache 클래스 정의
- [ ] __init__() - ttl, _cache 초기화
- [ ] get() - 만료 확인 후 반환
- [ ] set() - cached_at 포함하여 저장
- [ ] clear() - 전체/개별 삭제
- [ ] get_stats() - 통계 조회 (선택)

### processor.py
- [ ] DataProcessor 클래스 정의
- [ ] validate_and_clean() - 필수 필드/값 검증
- [ ] _clean_ohlcv() - None, 음수, 정렬 처리
- [ ] detect_price_spike() - 3σ 기준
- [ ] normalize_price() - minmax/zscore (선택)

### historical.py
- [ ] HistoricalDataLoader 클래스 정의
- [ ] __init__() - data_dir Path 변환
- [ ] load_historical_data() - CSV 로드
- [ ] 날짜 필터링 (pandas)
- [ ] _clean_historical_data() - 정제
- [ ] download_historical_data() - 스켈레톤 (TODO)

---

## 테스트 시나리오

### fetcher.py 테스트
```python
# 1. 초기화
exchange = ccxt.bybit({...})
fetcher = MarketDataFetcher(exchange)

# 2. 단일 심볼 수집
data = await fetcher.fetch_market_data('DOGE/USDT')
assert 'price' in data
assert len(data['ohlcv']) >= 50

# 3. 캐싱 확인
data2 = await fetcher.fetch_market_data('DOGE/USDT')
assert data == data2  # 캐시 히트

# 4. 다중 심볼
all_data = await fetcher.fetch_multiple_symbols()
assert 'DOGE/USDT' in all_data
assert 'SOL/USDT' in all_data
```

### cache.py 테스트
```python
# 1. 초기화
cache = DataCache(ttl=60)

# 2. 저장 및 조회
cache.set('DOGE/USDT', {'price': 0.3821})
data = cache.get('DOGE/USDT')
assert data['price'] == 0.3821

# 3. 만료 확인
import time
time.sleep(61)
expired = cache.get('DOGE/USDT')
assert expired is None

# 4. 통계
stats = cache.get_stats()
assert 'total_keys' in stats
```

### processor.py 테스트
```python
# 1. 검증 성공
processor = DataProcessor()
data = {
    'symbol': 'DOGE/USDT',
    'price': 0.3821,
    'ohlcv': [[...]] * 100,
    'timestamp': 1234567890
}
cleaned = processor.validate_and_clean(data)
assert cleaned['price'] == 0.3821

# 2. 검증 실패
try:
    bad_data = {'symbol': 'DOGE/USDT'}
    processor.validate_and_clean(bad_data)
except DataValidationError:
    pass  # 정상

# 3. 스파이크 감지
is_spike = processor.detect_price_spike(
    150.0,  # 급등
    [100.0] * 20
)
assert is_spike == True
```

### historical.py 테스트
```python
# 1. 초기화
loader = HistoricalDataLoader()

# 2. CSV 로드
data = loader.load_historical_data(
    'DOGE/USDT',
    '2024-01-01',
    '2024-01-31'
)
assert len(data) > 0
assert 'timestamp' in data[0]
assert 'close' in data[0]

# 3. 파일 없음
try:
    loader.load_historical_data(
        'INVALID/USDT',
        '2024-01-01',
        '2024-01-31'
    )
except DataFetchError:
    pass  # 정상
```

---

## 완성도 체크

### 이 명세서로 가능한 것
- ✅ 모든 클래스 구조 정확히 작성
- ✅ 모든 함수 시그니처 정확히 작성
- ✅ 어디서 호출되는지 명확
- ✅ 무엇을 반환하는지 명확
- ✅ 예외 처리 방법 명확
- ✅ 캐싱 로직 구현 가능
- ✅ async/await 처리 가능
- ✅ 테스트 케이스 작성 가능

### 추가 필요 사항
- ❓ CSV 파일 실제 다운로드 구현 (선택)
- ❓ 더 복잡한 데이터 검증 (선택)

---

## 주요 특징

### 1. 캐싱 시스템
- 1분간 유효
- 중복 API 호출 방지
- API 비용 절감

### 2. 데이터 검증
- 필수 필드 확인
- 가격 유효성 검증
- 최소 데이터 개수 확인

### 3. 에러 처리
- 외부 API 예외 래핑
- 명확한 에러 메시지
- 부분 실패 허용 (다중 심볼)

### 4. 비동기 처리
- async/await 사용
- ccxt 비동기 메서드 활용
- 효율적인 데이터 수집

---

## 다음 단계

DATA 모듈 완성 후:
1. **INDICATORS 모듈** (03_INDICATORS_함수명세서.md)
2. DATA의 OHLCV를 받아 지표 계산
3. RSI, MACD, Bollinger, Fibonacci 구현

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 2 (데이터 레이어)  
**검증**: ✅ 완료

이 명세서 + 기획서 = 동일한 DATA 모듈 개발 가능

---

### 📌 함수: DataProcessor.validate_and_clean(market_data)

```python
def validate_and_clean(self, market_data: Dict) -> Dict:
```

#### 역할
시장 데이터 검증 및 정제

#### 인자
- `market_data: Dict` - fetch_market_data()의 원본 반환값

#### 사용하는 모듈/함수
- `self._clean_ohlcv(ohlcv)`

#### 호출되는 곳
```python
# data/fetcher.py fetch_market_data()
market_data = self.processor.validate_and_clean(market_data)
```

#### 검증 항목
1. 필수 필드: symbol, price, ohlcv, timestamp
2. price > 0
3. OHLCV 최소 50개

#### 반환값
- `Dict`: 검증 및 정제된 market_data

#### 예외 처리
```python
- DataValidationError: 필수 필드 누락 또는 유효하지 않은 값
- InsufficientDataError: OHLCV < 50개
```

#### 구현 코드
```python
def validate_and_clean(self, market_data: Dict) -> Dict:
    """
    시장 데이터 검증 및 정제
    
    Args:
        market_data: 원본 시장 데이터
    
    Returns:
        정제된 시장 데이터
    
    Raises:
        DataValidationError: 검증 실패
    """
    # 필수 필드 확인
    required = ['symbol', 'price', 'ohlcv', 'timestamp']
    for field in required:
        if field not in market_data:
            raise DataValidationError(f"필수 필드 누락: {field}")
    
    # 가격 유효성
    if market_data['price'] <= 0:
        raise DataValidationError("가격이 0 이하")
    
    # OHLCV 최소 개수
    if len(market_data['ohlcv']) < 50:
        raise InsufficientDataError(
            f"OHLCV 데이터 부족: {len(market_data['ohlcv'])}/50"
        )
    
    # OHLCV 정제
    market_data['ohlcv'] = self._clean_ohlcv(market_data['ohlcv'])
    
    return market_data
```

---

### 📌 함수: DataProcessor._clean_ohlcv(ohlcv)

```python
def _clean_ohlcv(self, ohlcv: List[List]) -> List[List]:
```

#### 역할
OHLCV 데이터 정제 (내부 메서드)

#### 인자
- `ohlcv: List[List]` - [[timestamp, o, h, l, c, v], ...]

#### 호출되는 곳
- `self.validate_and_clean()` 내부에서만

#### 정제 작업
1. None 값 제거
2. 가격 ≤ 0 제거
3. High < Low 제거
4. 시간순 정렬

#### 반환값
- `List[List]`: 정제된 OHLCV

#### 구현 코드
```python
def _clean_ohlcv(self, ohlcv: List[List]) -> List[List]:
    """
    OHLCV 데이터 정제
    
    - None 값 제거
    - 이상치 제거 (가격 0 또는 음수)
    - 정렬
    """
    cleaned = []
    
    for candle in ohlcv:
        # None 체크
        if any(v is None for v in candle[:5]):
            continue
        
        timestamp, o, h, l, c, v = candle
        
        # 가격 유효성
        if any(p <= 0 for p in [o, h, l, c]):
            continue
        
        # High >= Low 확인
        if h < l:
            continue
        
        cleaned.append(candle)
    
    # 시간순 정렬
    cleaned.sort(key=lambda x: x[0])
    
    return cleaned
```

---

### 📌 함수: DataProcessor.detect_price_spike(current_price, recent_prices, threshold)

```python
def detect_price_spike(
    self,
    current_price: float,
    recent_prices: List[float],
    threshold: float = 3.0
) -> bool:
```

#### 역할
가격 스파이크 감지 (비정상적인 급등락)

#### 인자
- `current_price: float` - 현재가
- `recent_prices: List[float]` - 최근 20개 가격
- `threshold: float = 3.0` - 표준편차 배수

#### 사용하는 모듈/함수
- `numpy.mean()`
- `numpy.std()`

#### 호출되는 곳
```python
# strategy/entry.py check_entry_conditions()
recent_prices = [candle[4] for candle in ohlcv[-20:]]
is_spike = self.processor.detect_price_spike(
    current_price,
    recent_prices
)
if is_spike:
    return None  # 진입 중단

# engine/base_engine.py main_loop()
if processor.detect_price_spike(data['price'], recent_prices):
    logger.warning("가격 스파이크 감지, 거래 보류")
    continue
```

#### 로직
```python
1. recent_prices < 20개면 False 반환
2. 평균 = mean(recent_prices)
3. 표준편차 = std(recent_prices)
4. 상한 = 평균 + (3 × 표준편차)
5. 하한 = 평균 - (3 × 표준편차)
6. current_price가 상한/하한 벗어나면 True
```

#### 반환값
- `bool`:
  - True: 스파이크 감지 (거래 중단 권장)
  - False: 정상

#### 구현 코드
```python
def detect_price_spike(
    self,
    current_price: float,
    recent_prices: List[float],
    threshold: float = 3.0
) -> bool:
    """
    가격 스파이크 감지
    
    Args:
        current_price: 현재가
        recent_prices: 최근 20개 가격
        threshold: 표준편차 배수 (기본 3.0)
    
    Returns:
        True: 스파이크 