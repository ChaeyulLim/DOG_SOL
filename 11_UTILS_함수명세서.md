알겠습니다! 3개 파일 순서대로 만들겠습니다.

<artifact identifier="utils-functions-spec" type="text/markdown" title="11_UTILS_함수명세서.md">
# 11_UTILS 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [utils/network.py](#utilsnetworkpy)
2. [utils/fee_calculator.py](#utilsfee_calculatorpy)
3. [utils/validators.py](#utilsvalidatorspy)
4. [utils/helpers.py](#utilshelperspy)
5. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📄 utils/network.py

### 파일 전체 구조
```python
import asyncio
import logging
import time
from functools import wraps
from typing import Callable, Any
import aiohttp
import ccxt

logger = logging.getLogger(__name__)

def retry_on_network_error(
    max_retries: int = 60,
    delay: int = 1,
    exponential_backoff: bool = False
) -> Callable: ...

async def check_internet_connection() -> bool: ...

async def wait_for_connection(timeout: int = 300) -> bool: ...

class NetworkMonitor:
    def __init__(self): ...
    def record_request(self, success: bool) -> None: ...
    def get_success_rate(self) -> float: ...
    def is_stable(self) -> bool: ...
```

---

### 📌 함수: retry_on_network_error(max_retries, delay, exponential_backoff)

```python
def retry_on_network_error(
    max_retries: int = 60,
    delay: int = 1,
    exponential_backoff: bool = False
) -> Callable:
```

#### 역할
네트워크 오류 시 자동 재시도 데코레이터

#### 인자
- `max_retries: int = 60` - 최대 재시도 횟수
- `delay: int = 1` - 재시도 간격 (초)
- `exponential_backoff: bool = False` - 지수 백오프 사용

#### 재시도 대상 예외
```python
(
    ccxt.NetworkError,
    ccxt.RequestTimeout,
    aiohttp.ClientError,
    ConnectionError,
    TimeoutError
)
```

#### 호출되는 곳
```python
# exchanges/bybit_live.py
@retry_on_network_error(max_retries=60, delay=1)
async def fetch_ticker(self, symbol):
    return await self.exchange.fetch_ticker(symbol)
```

#### 구현 코드
```python
def retry_on_network_error(
    max_retries: int = 60,
    delay: int = 1,
    exponential_backoff: bool = False
) -> Callable:
    """
    네트워크 오류 시 자동 재시도
    
    Example:
        >>> @retry_on_network_error(max_retries=3, delay=2)
        >>> async def fetch_data():
        >>>     return await api.get()
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs) -> Any:
            last_exception = None
            
            for attempt in range(max_retries):
                try:
                    return await func(*args, **kwargs)
                
                except (
                    ccxt.NetworkError,
                    ccxt.RequestTimeout,
                    aiohttp.ClientError,
                    ConnectionError,
                    TimeoutError
                ) as e:
                    last_exception = e
                    
                    if attempt == max_retries - 1:
                        logger.critical(
                            f"🚨 {func.__name__} 네트워크 오류 "
                            f"{max_retries}회 실패: {e}"
                        )
                        raise Exception(
                            f"네트워크 실패 ({max_retries}회): {e}"
                        )
                    
                    if exponential_backoff:
                        wait_time = min(delay * (2 ** attempt), 60)
                    else:
                        wait_time = delay
                    
                    logger.warning(
                        f"⚠️ {func.__name__} 네트워크 오류 "
                        f"({attempt+1}/{max_retries}): {e} "
                        f"| {wait_time}초 후 재시도"
                    )
                    
                    await asyncio.sleep(wait_time)
                
                except Exception as e:
                    logger.error(f"❌ {func.__name__} 오류: {e}")
                    raise
            
            raise last_exception
        
        return wrapper
    return decorator
```

---

### 📌 함수: check_internet_connection()

```python
async def check_internet_connection() -> bool:
```

#### 역할
인터넷 연결 상태 확인

#### 테스트 URL
```python
[
    'https://www.google.com',
    'https://www.cloudflare.com',
    'https://1.1.1.1'
]
```

#### 구현 코드
```python
async def check_internet_connection() -> bool:
    """
    인터넷 연결 확인
    
    Returns:
        True: 연결 OK, False: 연결 불가
    """
    test_urls = [
        'https://www.google.com',
        'https://www.cloudflare.com',
        'https://1.1.1.1'
    ]
    
    timeout = aiohttp.ClientTimeout(total=3)
    
    try:
        async with aiohttp.ClientSession(timeout=timeout) as session:
            for url in test_urls:
                try:
                    async with session.get(url) as response:
                        if response.status == 200:
                            return True
                except:
                    continue
        return False
    
    except Exception as e:
        logger.error(f"연결 확인 오류: {e}")
        return False
```

---

### 📌 함수: wait_for_connection(timeout)

```python
async def wait_for_connection(timeout: int = 300) -> bool:
```

#### 역할
네트워크 복구 대기 (최대 timeout 초)

#### 구현 코드
```python
async def wait_for_connection(timeout: int = 300) -> bool:
    """
    네트워크 복구 대기
    
    Args:
        timeout: 최대 대기 시간 (초)
    
    Returns:
        True: 복구됨, False: 타임아웃
    """
    start_time = asyncio.get_event_loop().time()
    check_interval = 1
    
    logger.info(f"🔄 네트워크 복구 대기 중 (최대 {timeout}초)...")
    
    while True:
        if await check_internet_connection():
            elapsed = asyncio.get_event_loop().time() - start_time
            logger.info(f"✅ 네트워크 복구! ({elapsed:.1f}초)")
            return True
        
        elapsed = asyncio.get_event_loop().time() - start_time
        if elapsed >= timeout:
            logger.error(f"❌ 네트워크 복구 실패 ({timeout}초)")
            return False
        
        if int(elapsed) % 30 == 0 and elapsed > 0:
            remaining = timeout - elapsed
            logger.info(f"⏳ 대기 중... (남은 시간: {remaining:.0f}초)")
        
        await asyncio.sleep(check_interval)
```

---

### 📌 클래스: NetworkMonitor

#### 목적
네트워크 요청 성공률 모니터링

---

### 📌 함수: NetworkMonitor.__init__()

```python
def __init__(self):
```

#### 구현 코드
```python
def __init__(self):
    """네트워크 모니터 초기화"""
    self.total_requests = 0
    self.successful_requests = 0
    self.failed_requests = 0
    self.recent_results = []
    self.max_recent = 100
    self.last_check_time = time.time()
    
    logger.info("📡 네트워크 모니터 시작")
```

---

### 📌 함수: NetworkMonitor.record_request(success)

```python
def record_request(self, success: bool) -> None:
```

#### 구현 코드
```python
def record_request(self, success: bool) -> None:
    """
    요청 결과 기록
    
    Example:
        >>> monitor.record_request(True)
    """
    self.total_requests += 1
    
    if success:
        self.successful_requests += 1
    else:
        self.failed_requests += 1
    
    self.recent_results.append(success)
    if len(self.recent_results) > self.max_recent:
        self.recent_results.pop(0)
    
    self.last_check_time = time.time()
```

---

### 📌 함수: NetworkMonitor.get_success_rate()

```python
def get_success_rate(self) -> float:
```

#### 구현 코드
```python
def get_success_rate(self) -> float:
    """최근 100개 성공률 조회"""
    if not self.recent_results:
        return 1.0
    
    success_count = sum(self.recent_results)
    return success_count / len(self.recent_results)
```

---

### 📌 함수: NetworkMonitor.is_stable()

```python
def is_stable(self) -> bool:
```

#### 구현 코드
```python
def is_stable(self) -> bool:
    """
    네트워크 안정성 판단
    
    Returns:
        True: 안정 (95%+)
    """
    return self.get_success_rate() >= 0.95
```

---

## 📄 utils/fee_calculator.py

### 파일 전체 구조
```python
from typing import Dict

class FeeCalculator:
    def __init__(self, exchange_name: str = 'bybit'): ...
    
    def calculate_entry_fee(
        self,
        entry_price: float,
        quantity: float
    ) -> Dict: ...
    
    def calculate_exit_fee(
        self,
        exit_price: float,
        quantity: float
    ) -> Dict: ...
    
    def calculate_total_fees(
        self,
        entry_price: float,
        exit_price: float,
        quantity: float
    ) -> Dict: ...
    
    def calculate_net_pnl(
        self,
        entry_price: float,
        exit_price: float,
        quantity: float
    ) -> Dict: ...
    
    def get_breakeven_price(
        self,
        entry_price: float
    ) -> float: ...
```

---

### 📌 클래스: FeeCalculator

#### 목적
거래 수수료 계산 및 순수익 산출

---

### 📌 함수: FeeCalculator.__init__(exchange_name)

```python
def __init__(self, exchange_name: str = 'bybit'):
```

#### 수수료율
```python
FEE_RATES = {
    'bybit': {
        'maker': 0.001,  # 0.1%
        'taker': 0.001   # 0.1%
    }
}
```

#### 구현 코드
```python
def __init__(self, exchange_name: str = 'bybit'):
    """
    수수료 계산기 초기화
    
    Args:
        exchange_name: 거래소 이름
    """
    FEE_RATES = {
        'bybit': {'maker': 0.001, 'taker': 0.001},
        'binance': {'maker': 0.001, 'taker': 0.001}
    }
    
    if exchange_name not in FEE_RATES:
        raise ValueError(f"지원하지 않는 거래소: {exchange_name}")
    
    self.exchange_name = exchange_name
    self.fee_rates = FEE_RATES[exchange_name]
    self.fee_rate = self.fee_rates['taker']  # Market Order는 Taker
```

---

### 📌 함수: FeeCalculator.calculate_entry_fee(entry_price, quantity)

```python
def calculate_entry_fee(
    self,
    entry_price: float,
    quantity: float
) -> Dict:
```

#### 계산식
```
매수 금액 = entry_price × quantity
수수료 = 매수 금액 × 0.001
총 비용 = 매수 금액 + 수수료
```

#### 구현 코드
```python
def calculate_entry_fee(
    self,
    entry_price: float,
    quantity: float
) -> Dict:
    """
    진입 수수료 계산
    
    Returns:
        {
            'trade_value': 거래 금액,
            'fee': 수수료,
            'total_cost': 총 비용
        }
    """
    trade_value = entry_price * quantity
    fee = trade_value * self.fee_rate
    total_cost = trade_value + fee
    
    return {
        'trade_value': trade_value,
        'fee': fee,
        'total_cost': total_cost,
        'fee_rate': self.fee_rate
    }
```

---

### 📌 함수: FeeCalculator.calculate_exit_fee(exit_price, quantity)

```python
def calculate_exit_fee(
    self,
    exit_price: float,
    quantity: float
) -> Dict:
```

#### 구현 코드
```python
def calculate_exit_fee(
    self,
    exit_price: float,
    quantity: float
) -> Dict:
    """청산 수수료 계산"""
    trade_value = exit_price * quantity
    fee = trade_value * self.fee_rate
    net_revenue = trade_value - fee
    
    return {
        'trade_value': trade_value,
        'fee': fee,
        'net_revenue': net_revenue,
        'fee_rate': self.fee_rate
    }
```

---

### 📌 함수: FeeCalculator.calculate_total_fees(entry_price, exit_price, quantity)

```python
def calculate_total_fees(
    self,
    entry_price: float,
    exit_price: float,
    quantity: float
) -> Dict:
```

#### 구현 코드
```python
def calculate_total_fees(
    self,
    entry_price: float,
    exit_price: float,
    quantity: float
) -> Dict:
    """
    왕복 거래 총 수수료
    
    Returns:
        {
            'entry_fee': 진입 수수료,
            'exit_fee': 청산 수수료,
            'total_fee': 총 수수료,
            'total_fee_percent': 수수료율 (-0.2%)
        }
    """
    entry_info = self.calculate_entry_fee(entry_price, quantity)
    exit_info = self.calculate_exit_fee(exit_price, quantity)
    
    total_fee = entry_info['fee'] + exit_info['fee']
    total_fee_percent = total_fee / entry_info['total_cost']
    
    return {
        'entry_fee': entry_info['fee'],
        'exit_fee': exit_info['fee'],
        'total_fee': total_fee,
        'total_fee_percent': -total_fee_percent
    }
```

---

### 📌 함수: FeeCalculator.calculate_net_pnl(entry_price, exit_price, quantity)

```python
def calculate_net_pnl(
    self,
    entry_price: float,
    exit_price: float,
    quantity: float
) -> Dict:
```

#### 역할
수수료 포함 순손익 계산

#### 반환값
```python
{
    'gross_pnl': 2000.0,          # 명목 손익
    'gross_pnl_percent': 0.02,    # +2%
    'net_pnl': 1798.0,            # 순손익
    'net_pnl_percent': 0.01796,   # +1.796%
    'total_fee': 202.0,
    'fee_impact': -0.00204        # -0.204%
}
```

#### 구현 코드
```python
def calculate_net_pnl(
    self,
    entry_price: float,
    exit_price: float,
    quantity: float
) -> Dict:
    """수수료 포함 순손익"""
    entry_info = self.calculate_entry_fee(entry_price, quantity)
    exit_info = self.calculate_exit_fee(exit_price, quantity)
    
    total_fee = entry_info['fee'] + exit_info['fee']
    
    net_profit = exit_info['net_revenue'] - entry_info['total_cost']
    net_pnl_percent = net_profit / entry_info['total_cost']
    
    gross_profit = (exit_price - entry_price) * quantity
    gross_pnl_percent = (exit_price - entry_price) / entry_price
    
    fee_impact = net_pnl_percent - gross_pnl_percent
    
    return {
        'gross_pnl': gross_profit,
        'gross_pnl_percent': gross_pnl_percent,
        'net_pnl': net_profit,
        'net_pnl_percent': net_pnl_percent,
        'total_fee': total_fee,
        'fee_impact': fee_impact,
        'entry_cost': entry_info['total_cost'],
        'exit_revenue': exit_info['net_revenue']
    }
```

---

### 📌 함수: FeeCalculator.get_breakeven_price(entry_price)

```python
def get_breakeven_price(self, entry_price: float) -> float:
```

#### 역할
손익분기점 가격 (수수료 상쇄)

#### 계산식
```python
# 총 수수료: 0.2%
# 손익분기 = entry_price × 1.002
```

#### 구현 코드
```python
def get_breakeven_price(self, entry_price: float) -> float:
    """
    손익분기점 계산
    
    Example:
        >>> be = calc.get_breakeven_price(100.0)
        >>> print(f"손익분기: {be:.2f}")
        손익분기: 100.20
    """
    total_fee_rate = self.fee_rate * 2
    breakeven_price = entry_price * (1 + total_fee_rate)
    return breakeven_price
```

---

## 📄 utils/validators.py

### 파일 전체 구조
```python
from typing import Dict, Any
import re
from datetime import datetime

class DataValidator:
    @staticmethod
    def validate_symbol(symbol: str) -> bool: ...
    
    @staticmethod
    def validate_price(price: float) -> bool: ...
    
    @staticmethod
    def validate_quantity(quantity: float, min_qty: float = 0.001) -> bool: ...
    
    @staticmethod
    def validate_timestamp(timestamp: int) -> bool: ...
    
    @staticmethod
    def validate_ohlcv(ohlcv: list) -> bool: ...
    
    @staticmethod
    def validate_indicators(indicators: Dict) -> bool: ...
    
    @staticmethod
    def validate_ai_response(response: Dict) -> bool: ...
    
    @staticmethod
    def validate_trade_data(trade: Dict) -> bool: ...
```

---

### 📌 함수: DataValidator.validate_symbol(symbol)

```python
@staticmethod
def validate_symbol(symbol: str) -> bool:
```

#### 검증 규칙
```
패턴: [대문자]/[대문자]
예: DOGE/USDT, SOL/USDT
```

#### 구현 코드
```python
@staticmethod
def validate_symbol(symbol: str) -> bool:
    """
    심볼 형식 검증
    
    Example:
        >>> DataValidator.validate_symbol('DOGE/USDT')
        True
        >>> DataValidator.validate_symbol('doge/usdt')
        False
    """
    if not isinstance(symbol, str):
        return False
    
    pattern = r'^[A-Z]+/[A-Z]+$'
    return bool(re.match(pattern, symbol))
```

---

### 📌 함수: DataValidator.validate_price(price)

```python
@staticmethod
def validate_price(price: float) -> bool:
```

#### 검증 규칙
```
price > 0
price < 1,000,000 (비정상 가격 필터)
```

#### 구현 코드
```python
@staticmethod
def validate_price(price: float) -> bool:
    """
    가격 검증
    
    Example:
        >>> DataValidator.validate_price(0.3821)
        True
        >>> DataValidator.validate_price(-1.0)
        False
    """
    try:
        price = float(price)
        return 0 < price < 1_000_000
    except (TypeError, ValueError):
        return False
```

---

### 📌 함수: DataValidator.validate_quantity(quantity, min_qty)

```python
@staticmethod
def validate_quantity(quantity: float, min_qty: float = 0.001) -> bool:
```

#### 구현 코드
```python
@staticmethod
def validate_quantity(quantity: float, min_qty: float = 0.001) -> bool:
    """
    수량 검증
    
    Args:
        quantity: 수량
        min_qty: 최소 수량
    """
    try:
        quantity = float(quantity)
        return quantity >= min_qty
    except (TypeError, ValueError):
        return False
```

---

### 📌 함수: DataValidator.validate_timestamp(timestamp)

```python
@staticmethod
def validate_timestamp(timestamp: int) -> bool:
```

#### 검증 규칙
```
2020-01-01 < timestamp < 2030-12-31
```

#### 구현 코드
```python
@staticmethod
def validate_timestamp(timestamp: int) -> bool:
    """
    타임스탬프 검증
    
    Example:
        >>> DataValidator.validate_timestamp(1640000000)
        True
    """
    try:
        timestamp = int(timestamp)
        min_ts = 1577836800  # 2020-01-01
        max_ts = 1924991999  # 2030-12-31
        return min_ts <= timestamp <= max_ts
    except (TypeError, ValueError):
        return False
```

---

### 📌 함수: DataValidator.validate_ohlcv(ohlcv)

```python
@staticmethod
def validate_ohlcv(ohlcv: list) -> bool:
```

#### 검증 규칙
```python
[
    [timestamp, open, high, low, close, volume],
    ...
]
- 길이 6
- open, high, low, close > 0
- high >= low
- volume >= 0
```

#### 구현 코드
```python
@staticmethod
def validate_ohlcv(ohlcv: list) -> bool:
    """
    OHLCV 데이터 검증
    
    Example:
        >>> candle = [1640000000, 100, 105, 98, 102, 1000000]
        >>> DataValidator.validate_ohlcv([candle])
        True
    """
    if not isinstance(ohlcv, list) or len(ohlcv) == 0:
        return False
    
    for candle in ohlcv:
        if not isinstance(candle, list) or len(candle) != 6:
            return False
        
        try:
            ts, o, h, l, c, v = candle
            
            # 타임스탬프
            if not DataValidator.validate_timestamp(ts):
                return False
            
            # 가격
            if not all(DataValidator.validate_price(p) for p in [o, h, l, c]):
                return False
            
            # 논리 검증
            if h < l:  # high >= low
                return False
            
            # 거래량
            if v < 0:
                return False
        
        except (TypeError, ValueError):
            return False
    
    return True
```

---

### 📌 함수: DataValidator.validate_indicators(indicators)

```python
@staticmethod
def validate_indicators(indicators: Dict) -> bool:
```

#### 검증 규칙
```python
필수 키: ['rsi', 'macd', 'bollinger', 'fibonacci']
```

#### 구현 코드
```python
@staticmethod
def validate_indicators(indicators: Dict) -> bool:
    """
    지표 데이터 검증
    
    Example:
        >>> indicators = {
        >>>     'rsi': {'value': 45.2},
        >>>     'macd': {'value': 0.001},
        >>>     'bollinger': {'upper': 0.39},
        >>>     'fibonacci': {'support': 0.38}
        >>> }
        >>> DataValidator.validate_indicators(indicators)
        True
    """
    if not isinstance(indicators, dict):
        return False
    
    required_keys = ['rsi', 'macd', 'bollinger', 'fibonacci']
    
    for key in required_keys:
        if key not in indicators:
            return False
        
        if not isinstance(indicators[key], dict):
            return False
    
    return True
```

---

### 📌 함수: DataValidator.validate_ai_response(response)

```python
@staticmethod
def validate_ai_response(response: Dict) -> bool:
```

#### 검증 규칙
```python
필수 키: ['action', 'confidence', 'reasoning']
action: 'ENTER' | 'EXIT' | 'HOLD' | 'WAIT'
confidence: 0.0 ~ 1.0
```

#### 구현 코드
```python
@staticmethod
def validate_ai_response(response: Dict) -> bool:
    """
    AI 응답 검증
    
    Example:
        >>> response = {
        >>>     'action': 'ENTER',
        >>>     'confidence': 0.75,
        >>>     'reasoning': '...'
        >>> }
        >>> DataValidator.validate_ai_response(response)
        True
    """
    if not isinstance(response, dict):
        return False
    
    # 필수 키
    required = ['action', 'confidence', 'reasoning']
    if not all(key in response for key in required):
        return False
    
    # action 검증
    valid_actions = ['ENTER', 'EXIT', 'HOLD', 'WAIT']
    if response['action'] not in valid_actions:
        return False
    
    # confidence 검증
    try:
        conf = float(response['confidence'])
        if not (0 <= conf <= 1):
            return False
    except (TypeError, ValueError):
        return False
    
    # reasoning 검증
    if not isinstance(response['reasoning'], str):
        return False
    
    return True
```

---

### 📌 함수: DataValidator.validate_trade_data(trade)

```python
@staticmethod
def validate_trade_data(trade: Dict) -> bool:
```

#### 검증 규칙
```python
필수 키: ['symbol', 'entry_price', 'quantity', 'timestamp']
```

#### 구현 코드
```python
@staticmethod
def validate_trade_data(trade: Dict) -> bool:
    """
    거래 데이터 검증
    
    Example:
        >>> trade = {
        >>>     'symbol': 'DOGE/USDT',
        >>>     'entry_price': 0.3821,
        >>>     'quantity': 1000,
        >>>     'timestamp': 1640000000
        >>> }
        >>> DataValidator.validate_trade_data(trade)
        True
    """
    if not isinstance(trade, dict):
        return False
    
    required = ['symbol', 'entry_price', 'quantity', 'timestamp']
    if not all(key in trade for key in required):
        return False
    
    # 개별 검증
    if not DataValidator.validate_symbol(trade['symbol']):
        return False
    
    if not DataValidator.validate_price(trade['entry_price']):
        return False
    
    if not DataValidator.validate_quantity(trade['quantity']):
        return False
    
    if not DataValidator.validate_timestamp(trade['timestamp']):
        return False
    
    return True
```

---

## 📄 utils/helpers.py

### 파일 전체 구조
```python
from datetime import datetime, timedelta
from typing import Dict, List, Any
import json

def timestamp_to_datetime(timestamp: int) -> str: ...

def datetime_to_timestamp(dt_str: str) -> int: ...

def format_number(num: float, decimals: int = 2) -> str: ...

def format_percentage(value: float, decimals: int = 2) -> str: ...

def format_krw(amount: float) -> str: ...

def calculate_time_diff(start: int, end: int) -> str: ...

def safe_divide(a: float, b: float, default: float = 0.0) -> float: ...

def deep_get(dictionary: Dict, keys: str, default: Any = None) -> Any: ...

def truncate_string(text: str, max_length: int = 50) -> str: ...

def load_json_file(filepath: str) -> Dict: ...

def save_json_file(filepath: str, data: Dict) -> bool: ...
```

---

### 📌 함수: timestamp_to_datetime(timestamp)

```python
def timestamp_to_datetime(timestamp: int) -> str:
```

#### 구현 코드
```python
def timestamp_to_datetime(timestamp: int) -> str:
    """
    타임스탬프 → 날짜 문자열
    
    Example:
        >>> timestamp_to_datetime(1640000000)
        '2021-12-20 13:33:20'
    """
    return datetime.fromtimestamp(timestamp).strftime('%Y-%m-%d %H:%M:%S')
```

---

### 📌 함수:
```python
def datetime_to_timestamp(dt_str: str) -> int:
```

#### 구현 코드
```python
def datetime_to_timestamp(dt_str: str) -> int:
    """
    날짜 문자열 → 타임스탬프
    
    Example:
        >>> datetime_to_timestamp('2021-12-20 13:33:20')
        1640000000
    """
    dt = datetime.strptime(dt_str, '%Y-%m-%d %H:%M:%S')
    return int(dt.timestamp())
```

---

### 📌 함수: format_number(num, decimals)

```python
def format_number(num: float, decimals: int = 2) -> str:
```

#### 구현 코드
```python
def format_number(num: float, decimals: int = 2) -> str:
    """
    숫자 포맷팅 (천 단위 콤마)
    
    Example:
        >>> format_number(1234567.89, 2)
        '1,234,567.89'
    """
    return f"{num:,.{decimals}f}"
```

---

### 📌 함수: format_percentage(value, decimals)

```python
def format_percentage(value: float, decimals: int = 2) -> str:
```

#### 구현 코드
```python
def format_percentage(value: float, decimals: int = 2) -> str:
    """
    백분율 포맷팅
    
    Example:
        >>> format_percentage(0.0234, 2)
        '+2.34%'
        >>> format_percentage(-0.015, 2)
        '-1.50%'
    """
    return f"{value*100:+.{decimals}f}%"
```

---

### 📌 함수: format_krw(amount)

```python
def format_krw(amount: float) -> str:
```

#### 구현 코드
```python
def format_krw(amount: float) -> str:
    """
    KRW 금액 포맷팅
    
    Example:
        >>> format_krw(1234567)
        '1,234,567 KRW'
    """
    return f"{amount:,.0f} KRW"
```

---

### 📌 함수: calculate_time_diff(start, end)

```python
def calculate_time_diff(start: int, end: int) -> str:
```

#### 구현 코드
```python
def calculate_time_diff(start: int, end: int) -> str:
    """
    시간 차이 계산 (사람이 읽기 쉬운 형태)
    
    Example:
        >>> calculate_time_diff(1640000000, 1640007200)
        '2h 0m'
    """
    diff_seconds = end - start
    
    if diff_seconds < 60:
        return f"{diff_seconds:.0f}s"
    
    minutes = diff_seconds // 60
    if minutes < 60:
        return f"{minutes:.0f}m"
    
    hours = minutes // 60
    remaining_minutes = minutes % 60
    
    if hours < 24:
        return f"{hours:.0f}h {remaining_minutes:.0f}m"
    
    days = hours // 24
    remaining_hours = hours % 24
    return f"{days:.0f}d {remaining_hours:.0f}h"
```

---

### 📌 함수: safe_divide(a, b, default)

```python
def safe_divide(a: float, b: float, default: float = 0.0) -> float:
```

#### 구현 코드
```python
def safe_divide(a: float, b: float, default: float = 0.0) -> float:
    """
    안전한 나눗셈 (0으로 나누기 방지)
    
    Example:
        >>> safe_divide(10, 2)
        5.0
        >>> safe_divide(10, 0)
        0.0
        >>> safe_divide(10, 0, default=1.0)
        1.0
    """
    try:
        if b == 0:
            return default
        return a / b
    except (TypeError, ZeroDivisionError):
        return default
```

---

### 📌 함수: deep_get(dictionary, keys, default)

```python
def deep_get(dictionary: Dict, keys: str, default: Any = None) -> Any:
```

#### 구현 코드
```python
def deep_get(dictionary: Dict, keys: str, default: Any = None) -> Any:
    """
    중첩된 딕셔너리에서 안전하게 값 가져오기
    
    Args:
        dictionary: 대상 딕셔너리
        keys: 점(.)으로 구분된 키 경로
        default: 기본값
    
    Example:
        >>> data = {'a': {'b': {'c': 123}}}
        >>> deep_get(data, 'a.b.c')
        123
        >>> deep_get(data, 'a.b.d', default=0)
        0
    """
    try:
        for key in keys.split('.'):
            dictionary = dictionary[key]
        return dictionary
    except (KeyError, TypeError):
        return default
```

---

### 📌 함수: truncate_string(text, max_length)

```python
def truncate_string(text: str, max_length: int = 50) -> str:
```

#### 구현 코드
```python
def truncate_string(text: str, max_length: int = 50) -> str:
    """
    문자열 자르기
    
    Example:
        >>> truncate_string('This is a very long text', 10)
        'This is...'
    """
    if len(text) <= max_length:
        return text
    return text[:max_length-3] + '...'
```

---

### 📌 함수: load_json_file(filepath)

```python
def load_json_file(filepath: str) -> Dict:
```

#### 구현 코드
```python
def load_json_file(filepath: str) -> Dict:
    """
    JSON 파일 로드
    
    Example:
        >>> data = load_json_file('config.json')
    """
    try:
        with open(filepath, 'r', encoding='utf-8') as f:
            return json.load(f)
    except FileNotFoundError:
        return {}
    except json.JSONDecodeError:
        return {}
```

---

### 📌 함수: save_json_file(filepath, data)

```python
def save_json_file(filepath: str, data: Dict) -> bool:
```

#### 구현 코드
```python
def save_json_file(filepath: str, data: Dict) -> bool:
    """
    JSON 파일 저장
    
    Example:
        >>> save_json_file('config.json', {'key': 'value'})
        True
    """
    try:
        with open(filepath, 'w', encoding='utf-8') as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
        return True
    except Exception as e:
        print(f"JSON 저장 오류: {e}")
        return False
```

---

## 전체 의존성 그래프

### UTILS 모듈 구조
```
utils/
├── network.py (독립)
│   ├── 사용: asyncio, aiohttp, ccxt
│   └── 사용처: exchanges/, data/, engine/
│
├── fee_calculator.py (독립)
│   ├── 사용: core/constants
│   └── 사용처: engine/, monitoring/
│
├── validators.py (독립)
│   ├── 사용: re, datetime
│   └── 사용처: data/, ai/, exchanges/, engine/
│
└── helpers.py (독립)
    ├── 사용: datetime, json
    └── 사용처: 모든 모듈
```

### 사용하는 모듈
```
표준 라이브러리:
  - asyncio
  - logging
  - time
  - re
  - json
  - datetime

외부 라이브러리:
  - aiohttp
  - ccxt

내부 모듈:
  - core/constants (FEE_RATES)
```

### 사용되는 곳
```
exchanges/bybit_live.py
  └── @retry_on_network_error
  └── FeeCalculator
  └── DataValidator

data/fetcher.py
  └── @retry_on_network_error
  └── DataValidator

engine/base_engine.py
  └── NetworkMonitor
  └── FeeCalculator
  └── DataValidator
  └── helpers (전체)

monitoring/reporter.py
  └── format_number
  └── format_percentage
  └── format_krw

ai/analyzer.py
  └── DataValidator.validate_ai_response
```

---

## 개발 체크리스트

### network.py
- [ ] retry_on_network_error 데코레이터
- [ ] check_internet_connection 함수
- [ ] wait_for_connection 함수
- [ ] NetworkMonitor 클래스
  - [ ] record_request 메서드
  - [ ] get_success_rate 메서드
  - [ ] is_stable 메서드

### fee_calculator.py
- [ ] FeeCalculator 클래스
- [ ] calculate_entry_fee 메서드
- [ ] calculate_exit_fee 메서드
- [ ] calculate_total_fees 메서드
- [ ] calculate_net_pnl 메서드
- [ ] get_breakeven_price 메서드

### validators.py
- [ ] DataValidator 클래스
- [ ] validate_symbol 정적 메서드
- [ ] validate_price 정적 메서드
- [ ] validate_quantity 정적 메서드
- [ ] validate_timestamp 정적 메서드
- [ ] validate_ohlcv 정적 메서드
- [ ] validate_indicators 정적 메서드
- [ ] validate_ai_response 정적 메서드
- [ ] validate_trade_data 정적 메서드

### helpers.py
- [ ] timestamp_to_datetime 함수
- [ ] datetime_to_timestamp 함수
- [ ] format_number 함수
- [ ] format_percentage 함수
- [ ] format_krw 함수
- [ ] calculate_time_diff 함수
- [ ] safe_divide 함수
- [ ] deep_get 함수
- [ ] truncate_string 함수
- [ ] load_json_file 함수
- [ ] save_json_file 함수

---

## 테스트 시나리오

### network.py 테스트
```python
import asyncio
from utils.network import retry_on_network_error, NetworkMonitor

# 1. 데코레이터 테스트
@retry_on_network_error(max_retries=3, delay=1)
async def test_api_call():
    # 의도적 실패 후 성공 시뮬레이션
    pass

# 2. NetworkMonitor 테스트
monitor = NetworkMonitor()
monitor.record_request(True)
monitor.record_request(True)
monitor.record_request(False)
print(f"성공률: {monitor.get_success_rate()*100:.1f}%")
print(f"안정성: {monitor.is_stable()}")
```

### fee_calculator.py 테스트
```python
from utils.fee_calculator import FeeCalculator

calc = FeeCalculator('bybit')

# 진입 수수료
entry = calc.calculate_entry_fee(100.0, 1000)
print(f"진입 수수료: {entry['fee']:.2f} USDT")

# 순손익 계산
pnl = calc.calculate_net_pnl(100.0, 102.0, 1000)
print(f"명목 수익: {pnl['gross_pnl_percent']*100:.2f}%")
print(f"실제 수익: {pnl['net_pnl_percent']*100:.2f}%")
print(f"수수료 영향: {pnl['fee_impact']*100:.2f}%")

# 손익분기
be = calc.get_breakeven_price(100.0)
print(f"손익분기점: {be:.2f} USDT")
```

### validators.py 테스트
```python
from utils.validators import DataValidator

# 심볼 검증
assert DataValidator.validate_symbol('DOGE/USDT') == True
assert DataValidator.validate_symbol('doge/usdt') == False

# 가격 검증
assert DataValidator.validate_price(0.3821) == True
assert DataValidator.validate_price(-1.0) == False

# AI 응답 검증
ai_response = {
    'action': 'ENTER',
    'confidence': 0.75,
    'reasoning': 'MACD golden cross'
}
assert DataValidator.validate_ai_response(ai_response) == True
```

### helpers.py 테스트
```python
from utils.helpers import *

# 날짜 변환
ts = 1640000000
dt_str = timestamp_to_datetime(ts)
print(f"날짜: {dt_str}")

# 포맷팅
print(format_number(1234567.89))  # 1,234,567.89
print(format_percentage(0.0234))  # +2.34%
print(format_krw(1000000))        # 1,000,000 KRW

# 시간 차이
diff = calculate_time_diff(1640000000, 1640007200)
print(f"경과 시간: {diff}")  # 2h 0m

# 안전한 나눗셈
result = safe_divide(10, 0, default=1.0)
print(f"결과: {result}")  # 1.0

# 중첩 딕셔너리
data = {'a': {'b': {'c': 123}}}
value = deep_get(data, 'a.b.c')
print(f"값: {value}")  # 123
```

---

## 주요 특징

### 1. 네트워크 안정성
- 자동 재시도 (최대 60회)
- 지수 백오프 옵션
- 연결 상태 모니터링
- 복구 대기 메커니즘

### 2. 정확한 수수료 계산
- 거래소별 수수료율 지원
- 진입/청산 수수료 분리
- 순손익 자동 계산
- 손익분기점 제공

### 3. 철저한 데이터 검증
- 입력 데이터 형식 검증
- 논리적 무결성 확인
- 범위 검증
- 타입 안전성

### 4. 편리한 헬퍼 함수
- 날짜/시간 변환
- 숫자 포맷팅
- 안전한 연산
- JSON 처리

---

## 통합 사용 예시

### engine/base_engine.py에서 사용
```python
from utils.network import retry_on_network_error, NetworkMonitor
from utils.fee_calculator import FeeCalculator
from utils.validators import DataValidator
from utils.helpers import *

class BaseEngine:
    def __init__(self, config, exchange):
        # 네트워크 모니터
        self.network_monitor = NetworkMonitor()
        
        # 수수료 계산기
        self.fee_calculator = FeeCalculator('bybit')
        
        # 검증기
        self.validator = DataValidator()
    
    @retry_on_network_error(max_retries=60, delay=1)
    async def fetch_data(self, symbol):
        # 심볼 검증
        if not self.validator.validate_symbol(symbol):
            raise ValueError(f"잘못된 심볼: {symbol}")
        
        # 데이터 조회
        data = await self.exchange.fetch_ticker(symbol)
        self.network_monitor.record_request(True)
        
        # 가격 검증
        if not self.validator.validate_price(data['last']):
            raise ValueError(f"비정상 가격: {data['last']}")
        
        return data
    
    async def execute_exit(self, symbol, position):
        # 순손익 계산
        pnl_info = self.fee_calculator.calculate_net_pnl(
            position['entry_price'],
            current_price,
            position['quantity']
        )
        
        # 로그
        logger.info(
            f"청산: {symbol} | "
            f"명목 수익: {format_percentage(pnl_info['gross_pnl_percent'])} | "
            f"실제 수익: {format_percentage(pnl_info['net_pnl_percent'])} | "
            f"수수료: {format_krw(pnl_info['total_fee'] * 1300)}"
        )
```

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 11 (유틸리티 레이어)  
**검증**: ✅ 완료
</artifact identifier="utils-functions-spec">
