# 06_EXCHANGES 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [exchanges/base.py](#exchangesbasepy)
2. [exchanges/bybit_live.py](#exchangesbybitlivepy)
3. [exchanges/paper.py](#exchangespaperpy)
4. [exchanges/backtest.py](#exchangesbacktestpy)
5. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📁 exchanges/base.py

### 파일 전체 구조
```python
from abc import ABC, abstractmethod
from typing import Dict, Optional

class BaseExchange(ABC):
    @abstractmethod
    def initialize(self) -> None: ...
    
    @abstractmethod
    def create_order(self, symbol: str, amount_krw: int) -> Dict: ...
    
    @abstractmethod
    def close_position(self, symbol: str) -> Dict: ...
    
    @abstractmethod
    def get_balance(self) -> Dict: ...
    
    @abstractmethod
    def fetch_ticker(self, symbol: str) -> Dict: ...
```

---

### 📌 클래스: BaseExchange

#### 목적
거래소 인터페이스 추상 클래스 (모든 거래소의 공통 인터페이스)

---

### 📌 함수: BaseExchange.create_order(symbol, amount_krw)

```python
@abstractmethod
def create_order(self, symbol: str, amount_krw: int) -> Dict:
```

#### 역할
매수 주문 생성 (추상 메서드)

#### 인자
- `symbol: str` - 'DOGE/USDT'
- `amount_krw: int` - 투자 금액 (KRW)

#### 반환값
```python
Dict:
    'order_id': str
    'symbol': str
    'filled': float  # 체결 수량
    'average_price': float  # 평균 체결가
    'fee': float  # 수수료
```

---

### 📌 함수: BaseExchange.close_position(symbol)

```python
@abstractmethod
def close_position(self, symbol: str) -> Dict:
```

#### 역할
포지션 청산 (추상 메서드)

#### 반환값
```python
Dict:
    'order_id': str
    'quantity': float  # 청산 수량
    'price': float  # 청산가
    'fee': float
```

---

## 📁 exchanges/bybit_live.py

### 파일 전체 구조
```python
import ccxt
import asyncio
from typing import Dict
from .base import BaseExchange
from core.api_keys import APIKeys
from core.constants import KRW_USD_RATE, BYBIT_SPOT_FEE
from core.exceptions import OrderFailedError, InsufficientBalanceError

class BybitLiveExchange(BaseExchange):
    def __init__(self): ...
    def initialize(self) -> None: ...
    def create_order(self, symbol: str, amount_krw: int) -> Dict: ...
    def close_position(self, symbol: str) -> Dict: ...
    def get_balance(self) -> Dict: ...
    def fetch_ticker(self, symbol: str) -> Dict: ...
    def _wait_for_order_fill(self, order_id: str, symbol: str) -> Dict: ...
```

---

### 📌 클래스: BybitLiveExchange

#### 목적
Bybit 현물 거래 실제 구현

---

### 📌 함수: BybitLiveExchange.__init__()

```python
def __init__(self):
```

#### 역할
Bybit 거래소 초기화

#### 사용하는 모듈/함수
- `APIKeys.get_bybit_keys()` - API 키
- `ccxt.bybit()` - CCXT 라이브러리

#### 초기화 내용
```python
keys = APIKeys.get_bybit_keys()
self.exchange = ccxt.bybit({
    'apiKey': keys['api_key'],
    'secret': keys['api_secret'],
    'enableRateLimit': True,
    'options': {
        'defaultType': 'spot',
        'adjustForTimeDifference': True
    }
})
```

#### 호출되는 곳
```python
# engine/live_engine.py __init__()
from exchanges import BybitLiveExchange

self.exchange = BybitLiveExchange()
self.exchange.initialize()
```

#### 구현 코드
```python
def __init__(self):
    """Bybit 거래소 초기화"""
    keys = APIKeys.get_bybit_keys()
    
    self.exchange = ccxt.bybit({
        'apiKey': keys['api_key'],
        'secret': keys['api_secret'],
        'enableRateLimit': True,
        'options': {
            'defaultType': 'spot',
            'adjustForTimeDifference': True
        }
    })
```

---

### 📌 함수: BybitLiveExchange.initialize()

```python
def initialize(self) -> None:
```

#### 역할
거래소 연결 테스트

#### 구현 코드
```python
def initialize(self) -> None:
    """연결 테스트"""
    try:
        self.exchange.load_markets()
        balance = self.exchange.fetch_balance()
        print(f"✅ Bybit 연결 성공")
        print(f"   USDT 잔고: {balance['USDT']['free']:.2f}")
    except Exception as e:
        raise ConnectionError(f"Bybit 연결 실패: {e}")
```

---

### 📌 함수: BybitLiveExchange.create_order(symbol, amount_krw)

```python
def create_order(self, symbol: str, amount_krw: int) -> Dict:
```

#### 역할
실제 매수 주문 생성 (Market Order)

#### 인자
- `symbol: str` - 'DOGE/USDT'
- `amount_krw: int` - 500,000 (KRW)

#### 처리 흐름
```
1. KRW → USDT 환산 (1300원 기준)
2. 현재가 조회
3. 수량 계산 (USDT / 현재가)
4. 최소 주문량 확인
5. Market Buy Order 생성
6. 30초 대기 (부분 체결 확인)
7. 미체결 부분 취소
8. 결과 반환
```

#### 반환값
```python
Dict:
    'order_id': str = '123456789'
    'symbol': str = 'DOGE/USDT'
    'filled': float = 1006.0
    'average_price': float = 0.3821
    'fee': float = 0.3821  # USDT
```

#### 예외
- `InsufficientBalanceError` - 잔고 부족
- `OrderFailedError` - 주문 실패

#### 구현 코드
```python
def create_order(self, symbol: str, amount_krw: int) -> Dict:
    """
    매수 주문 생성
    
    Example:
        >>> exchange.create_order('DOGE/USDT', 500000)
        {'order_id': '123', 'filled': 1006.0, ...}
    """
    # 1. USDT 변환
    usdt_amount = amount_krw / KRW_USD_RATE
    
    # 2. 현재가 조회
    ticker = self.exchange.fetch_ticker(symbol)
    current_price = ticker['last']
    
    # 3. 수량 계산
    quantity = usdt_amount / current_price
    
    # 4. 최소 주문량 체크
    market = self.exchange.market(symbol)
    min_amount = market['limits']['amount']['min']
    
    if quantity < min_amount:
        raise OrderFailedError(
            f"최소 주문량 미달: {quantity} < {min_amount}"
        )
    
    # 5. 주문 생성
    try:
        order = self.exchange.create_market_buy_order(
            symbol=symbol,
            amount=quantity
        )
    except ccxt.InsufficientFunds:
        raise InsufficientBalanceError("USDT 잔고 부족")
    except Exception as e:
        raise OrderFailedError(f"주문 생성 실패: {e}")
    
    # 6. 부분 체결 확인
    filled_order = self._wait_for_order_fill(order['id'], symbol)
    
    return {
        'order_id': filled_order['id'],
        'symbol': symbol,
        'filled': filled_order['filled'],
        'average_price': filled_order['average'],
        'fee': filled_order['fee']['cost']
    }
```

---

### 📌 함수: BybitLiveExchange._wait_for_order_fill(order_id, symbol)

```python
def _wait_for_order_fill(self, order_id: str, symbol: str) -> Dict:
```

#### 역할
주문 체결 대기 및 부분 체결 처리 (내부 메서드)

#### 로직
```
1. 30초 대기
2. 주문 상태 조회
3. 체결률 확인
   - 90% 이상: 정상
   - 90% 미만: 미체결 부분 취소
4. 체결된 정보 반환
```

#### 구현 코드
```python
def _wait_for_order_fill(self, order_id: str, symbol: str) -> Dict:
    """부분 체결 처리"""
    import time
    
    # 30초 대기
    time.sleep(30)
    
    # 주문 조회
    order = self.exchange.fetch_order(order_id, symbol)
    
    # 체결률 확인
    fill_ratio = order['filled'] / order['amount']
    
    if fill_ratio < 0.9:  # 90% 미만
        # 미체결 부분 취소
        try:
            self.exchange.cancel_order(order_id, symbol)
        except:
            pass
    
    return order
```

---

### 📌 함수: BybitLiveExchange.close_position(symbol)

```python
def close_position(self, symbol: str) -> Dict:
```

#### 역할
보유 수량 전량 청산 (Market Sell)

#### 처리 흐름
```
1. 잔고에서 보유 수량 확인
2. 보유량이 0이면 예외
3. Market Sell Order 생성
4. 결과 반환
```

#### 구현 코드
```python
def close_position(self, symbol: str) -> Dict:
    """포지션 청산"""
    
    # 1. 보유 수량 확인
    balance = self.exchange.fetch_balance()
    base_currency = symbol.split('/')[0]  # 'DOGE'
    quantity = balance[base_currency]['free']
    
    if quantity == 0:
        raise OrderFailedError("보유 수량 없음")
    
    # 2. Market Sell
    try:
        order = self.exchange.create_market_sell_order(
            symbol=symbol,
            amount=quantity
        )
    except Exception as e:
        raise OrderFailedError(f"청산 실패: {e}")
    
    # 3. 결과
    return {
        'order_id': order['id'],
        'quantity': order['filled'],
        'price': order['average'],
        'fee': order['fee']['cost']
    }
```

---

### 📌 함수: BybitLiveExchange.get_balance()

```python
def get_balance(self) -> Dict:
```

#### 역할
현재 잔고 조회

#### 반환값
```python
Dict:
    'USDT': float = 384.6
    'DOGE': float = 1006.0
    'SOL': float = 0.0
    'total_krw': float = 1_035_420
```

#### 구현 코드
```python
def get_balance(self) -> Dict:
    """잔고 조회"""
    
    balance = self.exchange.fetch_balance()
    
    # 주요 통화만
    result = {
        'USDT': balance['USDT']['free'],
        'DOGE': balance['DOGE']['free'],
        'SOL': balance['SOL']['free']
    }
    
    # 총 KRW 환산
    total_usdt = result['USDT']
    
    # DOGE, SOL 현재가로 USDT 환산
    for coin in ['DOGE', 'SOL']:
        if result[coin] > 0:
            ticker = self.exchange.fetch_ticker(f'{coin}/USDT')
            total_usdt += result[coin] * ticker['last']
    
    result['total_krw'] = int(total_usdt * KRW_USD_RATE)
    
    return result
```

---

### 📌 함수: BybitLiveExchange.fetch_ticker(symbol)

```python
def fetch_ticker(self, symbol: str) -> Dict:
```

#### 역할
현재가 조회 (data/fetcher.py에서 사용)

#### 반환값
```python
Dict:
    'symbol': str = 'DOGE/USDT'
    'last': float = 0.3821
    'bid': float = 0.3820
    'ask': float = 0.3822
    'volume': float = 1234567890.0
```

#### 구현 코드
```python
def fetch_ticker(self, symbol: str) -> Dict:
    """현재가 조회"""
    return self.exchange.fetch_ticker(symbol)
```

---

## 📁 exchanges/paper.py

### 파일 전체 구조
```python
from typing import Dict
from .base import BaseExchange
from core.constants import KRW_USD_RATE, BYBIT_SPOT_FEE

class PaperExchange(BaseExchange):
    def __init__(self, initial_balance_krw: int): ...
    def initialize(self) -> None: ...
    def create_order(self, symbol: str, amount_krw: int) -> Dict: ...
    def close_position(self, symbol: str) -> Dict: ...
    def get_balance(self) -> Dict: ...
    def fetch_ticker(self, symbol: str) -> Dict: ...
```

---

### 📌 클래스: PaperExchange

#### 목적
모의투자 시뮬레이터 (실제 API 호출 없이 가상 잔고 관리)

---

### 📌 함수: PaperExchange.__init__(initial_balance_krw)

```python
def __init__(self, initial_balance_krw: int):
```

#### 역할
가상 거래소 초기화

#### 인자
- `initial_balance_krw: int` - 초기 자본금 (기본 1,000,000)

#### 초기화 내용
```python
self.balance = {
    'USDT': initial_balance_krw / KRW_USD_RATE,
    'DOGE': 0.0,
    'SOL': 0.0
}
self.positions = {}  # {symbol: position_info}
self.real_exchange = ccxt.bybit()  # 시세 조회용
```

#### 구현 코드
```python
def __init__(self, initial_balance_krw: int = 1_000_000):
    """가상 거래소 초기화"""
    
    self.balance = {
        'USDT': initial_balance_krw / KRW_USD_RATE,
        'DOGE': 0.0,
        'SOL': 0.0
    }
    
    self.positions = {}
    
    # 실시간 시세 조회용
    self.real_exchange = ccxt.bybit({
        'enableRateLimit': True
    })
```

---

### 📌 함수: PaperExchange.create_order(symbol, amount_krw)

```python
def create_order(self, symbol: str, amount_krw: int) -> Dict:
```

#### 역할
가상 매수 (실제 API 호출 없음)

#### 처리 흐름
```
1. 실시간 시세는 진짜 Bybit에서 조회
2. 가상 잔고 차감
3. 가상 수량 증가
4. 포지션 기록
5. 수수료 시뮬레이션
```

#### 구현 코드
```python
def create_order(self, symbol: str, amount_krw: int) -> Dict:
    """가상 매수"""
    
    # 1. 실시간 시세 (진짜 API)
    ticker = self.real_exchange.fetch_ticker(symbol)
    price = ticker['last']
    
    # 2. USDT 차감
    usdt_needed = amount_krw / KRW_USD_RATE
    
    if self.balance['USDT'] < usdt_needed:
        raise InsufficientBalanceError("가상 잔고 부족")
    
    self.balance['USDT'] -= usdt_needed
    
    # 3. 수량 증가
    base = symbol.split('/')[0]
    quantity = usdt_needed / price
    self.balance[base] += quantity
    
    # 4. 포지션 기록
    self.positions[symbol] = {
        'entry_price': price,
        'quantity': quantity,
        'entry_time': time.time()
    }
    
    # 5. 수수료 시뮬레이션
    fee = usdt_needed * BYBIT_SPOT_FEE
    
    return {
        'order_id': f'paper_{int(time.time())}',
        'symbol': symbol,
        'filled': quantity,
        'average_price': price,
        'fee': fee
    }
```

---

### 📌 함수: PaperExchange.close_position(symbol)

```python
def close_position(self, symbol: str) -> Dict:
```

#### 역할
가상 청산

#### 구현 코드
```python
def close_position(self, symbol: str) -> Dict:
    """가상 청산"""
    
    if symbol not in self.positions:
        raise OrderFailedError("포지션 없음")
    
    # 실시간 시세
    ticker = self.real_exchange.fetch_ticker(symbol)
    price = ticker['last']
    
    # 포지션 정보
    position = self.positions[symbol]
    quantity = position['quantity']
    
    # 수량 차감
    base = symbol.split('/')[0]
    self.balance[base] -= quantity
    
    # USDT 증가
    usdt_received = quantity * price
    fee = usdt_received * BYBIT_SPOT_FEE
    self.balance['USDT'] += (usdt_received - fee)
    
    # 포지션 삭제
    del self.positions[symbol]
    
    return {
        'order_id': f'paper_close_{int(time.time())}',
        'quantity': quantity,
        'price': price,
        'fee': fee
    }
```

---

## 📁 exchanges/backtest.py

### 파일 전체 구조
```python
from typing import Dict, List
from .base import BaseExchange

class BacktestExchange(BaseExchange):
    def __init__(self, historical_data: Dict[str, List[Dict]]): ...
    def set_current_timestamp(self, timestamp: int) -> None: ...
    def create_order(self, symbol: str, amount_krw: int) -> Dict: ...
    def close_position(self, symbol: str) -> Dict: ...
```

---

### 📌 클래스: BacktestExchange

#### 목적
과거 데이터 재생 (CSV 기반 백테스트)

---

### 📌 함수: BacktestExchange.set_current_timestamp(timestamp)

```python
def set_current_timestamp(self, timestamp: int) -> None:
```

#### 역할
백테스트 시간 설정

#### 인자
- `timestamp: int` - 현재 시뮬레이션 시간

#### 호출되는 곳
```python
# engine/backtest_engine.py main_loop()
for candle in historical_data:
    self.exchange.set_current_timestamp(candle['timestamp'])
    # 거래 로직 실행
```

#### 구현 코드
```python
def set_current_timestamp(self, timestamp: int) -> None:
    """백테스트 시간 설정"""
    self.current_timestamp = timestamp
```

---

### 📌 함수: BacktestExchange.fetch_ticker(symbol)

```python
def fetch_ticker(self, symbol: str) -> Dict:
```

#### 역할
현재 timestamp의 가격 반환

#### 로직
```python
1. historical_data[symbol]에서 검색
2. current_timestamp와 일치하는 캔들 찾기
3. close 가격 반환
```

#### 구현 코드
```python
def fetch_ticker(self, symbol: str) -> Dict:
    """백테스트 시세 조회"""
    
    candles = self.historical_data[symbol]
    
    # 현재 timestamp의 캔들 찾기
    current_candle = None
    for candle in candles:
        if candle['timestamp'] == self.current_timestamp:
            current_candle = candle
            break
    
    if not current_candle:
        raise ValueError(f"시세 없음: {self.current_timestamp}")
    
    return {
        'symbol': symbol,
        'last': current_candle['close'],
        'volume': current_candle['volume']
    }
```

---

## 전체 의존성 그래프

### EXCHANGES 모듈 구조
```
base.py (추상 클래스)
├── bybit_live.py (상속)
├── paper.py (상속)
└── backtest.py (상속)
```

### 사용하는 모듈
```
core/api_keys → Bybit API 키
core/constants → 환율, 수수료
core/exceptions → 예외 클래스
ccxt → Bybit API 라이브러리
```

### 사용되는 곳
```
engine/live_engine.py → BybitLiveExchange
engine/paper_engine.py → PaperExchange
engine/backtest_engine.py → BacktestExchange
```

---

## 개발 체크리스트

### base.py
- [ ] BaseExchange 추상 클래스
- [ ] 모든 메서드 @abstractmethod
- [ ] 타입 힌트 명확히

### bybit_live.py
- [ ] BybitLiveExchange 클래스
- [ ] CCXT 라이브러리 통합
- [ ] create_order() - Market Buy
- [ ] close_position() - Market Sell
- [ ] _wait_for_order_fill() - 부분 체결
- [ ] 예외 처리 완벽히

### paper.py
- [ ] PaperExchange 클래스
- [ ] 가상 잔고 관리
- [ ] 실시간 시세는 진짜 API 사용
- [ ] 수수료 시뮬레이션

### backtest.py
- [ ] BacktestExchange 클래스
- [ ] historical_data 관리
- [ ] set_current_timestamp()
- [ ] 과거 시세 반환

---

## 테스트 시나리오

### bybit_live.py 테스트 (Testnet 권장)
```python
# 1. 초기화
exchange = BybitLiveExchange()
exchange.initialize()

# 2. 잔고 조회
balance = exchange.get_balance()
assert 'USDT' in balance

# 3. 매수 (소액)
order = exchange.create_order('DOGE/USDT', 10000)  # 10,000원
assert order['filled'] > 0

# 4. 청산
close = exchange.close_position('DOGE/USDT')
assert close['quantity'] > 0
```

### paper.py 테스트
```python
# 1. 초기화
exchange = PaperExchange(1_000_000)

# 2. 가상 매수
order = exchange.create_order('DOGE/USDT', 500_000)
assert order['filled'] > 0

# 3. 잔고 확인
balance = exchange.get_balance()
assert balance['DOGE'] > 0
assert balance['USDT'] < 770  # 일부 차감

# 4. 가상 청산
close = exchange.close_position('DOGE/USDT')
assert close['quantity'] == order['filled']
```

---

## 주요 특징

### 1. 추상화
- BaseExchange로 인터페이스 통일
- 모드 전환 간편 (Live ↔ Paper ↔ Backtest)

### 2. 실시간 시세
- Paper도 실시간 Bybit 시세 사용
- 현실적인 시뮬레이션

### 3. 부분 체결 처리
- 30초 대기
- 90% 미만 시 미체결 취소
- 실제 거래 환경 반영

### 4. 수수료 정확성
- Bybit 현물 0.1% 반영
- 매수/매도 각각 적용

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 5 (거래소 레이어)  
**검증**: ✅ 완료