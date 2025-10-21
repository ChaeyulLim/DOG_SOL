# 01_CORE 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. core/config.py
2. core/api_keys.py
3. core/constants.py
4. core/exceptions.py
5. 전체 의존성 그래프

---

## 📁 core/config.py

### 파일 전체 구조
```python
from dataclasses import dataclass
from typing import Dict

@dataclass
class Config:
    # 속성 정의
    # ...
    
    def __post_init__(self): ...
    
    @classmethod
    def load(cls, mode: str) -> 'Config': ...
    
    def get_position_size(self, symbol: str) -> int: ...
    
    def to_dict(self) -> Dict: ...
```

---

### 📌 클래스: Config

#### 목적
시스템 전체 설정값 중앙 관리

#### 속성 (Attributes)
```python
# 투자 설정
INVESTMENT_AMOUNT: int = 1_000_000
POSITION_ALLOCATION: Dict[str, float] = None

# 손익 구조
TAKE_PROFIT: float = 0.02
STOP_LOSS: float = -0.01
TRAILING_ACTIVATION: float = 0.005
TRAILING_STOP: float = -0.01

# 리스크 한도
DAILY_LOSS_LIMIT: float = -0.05
MONTHLY_DD_LIMIT: float = -0.10
CONSECUTIVE_LOSS_LIMIT: int = 3

# AI 설정
MIN_AI_CONFIDENCE: float = 0.70
AI_CHECK_INTERVAL: int = 7200

# 거래 설정
MAX_HOLDING_TIME: int = 86400
ORDER_TIMEOUT: int = 30
LOOP_INTERVAL: int = 60

# 모드
MODE: str = 'paper'
```

---

### 📌 함수: Config.__post_init__(self)

```python
def __post_init__(self):
```

#### 역할
dataclass 초기화 직후 자동 실행. POSITION_ALLOCATION 기본값 설정

#### 인자
- 없음 (self만)

#### 사용하는 모듈/함수
- 없음

#### 호출되는 곳
- dataclass 인스턴스 생성 시 자동 호출

#### 로직
```python
if self.POSITION_ALLOCATION is None:
    self.POSITION_ALLOCATION = {
        'DOGE': 0.5,
        'SOL': 0.5
    }
```

#### 반환값
- 없음 (self 수정)

#### 구현 코드
```python
def __post_init__(self):
    """초기화 후 기본값 설정"""
    if self.POSITION_ALLOCATION is None:
        self.POSITION_ALLOCATION = {
            'DOGE': 0.5,
            'SOL': 0.5
        }
```

---

### 📌 함수: Config.load(mode)

```python
@classmethod
def load(cls, mode: str = 'paper') -> 'Config':
```

#### 역할
모드에 따라 설정을 로드하는 팩토리 메서드

#### 인자
- mode: str = 'paper'
  - 가능한 값: 'paper', 'live', 'backtest'
  - 기본값: 'paper'

#### 사용하는 모듈/함수
- cls() - Config 인스턴스 생성

#### 호출되는 곳
```python
# run_paper.py
config = Config.load('paper')

# run_live.py
config = Config.load('live')

# run_backtest.py
config = Config.load('backtest')
```

#### 로직
1. 기본 Config 인스턴스 생성
2. MODE 속성 설정
3. 모드별 추가 설정:
   - 'live': MIN_AI_CONFIDENCE = 0.75
   - 'backtest': MIN_AI_CONFIDENCE = 1.0
   - 'paper': 기본값 유지

#### 반환값
- Config 인스턴스

#### 구현 코드
```python
@classmethod
def load(cls, mode: str = 'paper') -> 'Config':
    """모드에 따라 설정 로드"""
    config = cls()
    config.MODE = mode
    
    if mode == 'live':
        config.MIN_AI_CONFIDENCE = 0.75
    elif mode == 'backtest':
        config.MIN_AI_CONFIDENCE = 1.0
    
    return config
```

---

### 📌 함수: Config.get_position_size(symbol)

```python
def get_position_size(self, symbol: str) -> int:
```

#### 역할
심볼별 할당 금액(KRW) 계산

#### 인자
- symbol: str - 'DOGE' 또는 'SOL'

#### 사용하는 모듈/함수
- self.INVESTMENT_AMOUNT
- self.POSITION_ALLOCATION.get(symbol, 0)

#### 호출되는 곳
```python
# exchanges/bybit_live.py
amount_krw = self.config.get_position_size('DOGE')

# exchanges/paper.py
amount_krw = self.config.get_position_size('SOL')
```

#### 로직
```python
1. POSITION_ALLOCATION에서 비율 조회
2. INVESTMENT_AMOUNT × 비율
3. int()로 변환
```

#### 반환값
- int: 할당 금액 (KRW)
- 예: 1,000,000 × 0.5 = 500,000

#### 구현 코드
```python
def get_position_size(self, symbol: str) -> int:
    """심볼별 포지션 크기 계산"""
    allocation = self.POSITION_ALLOCATION.get(symbol, 0)
    return int(self.INVESTMENT_AMOUNT * allocation)
```

---

### 📌 함수: Config.to_dict()

```python
def to_dict(self) -> Dict:
```

#### 역할
설정을 딕셔너리로 변환 (로깅/저장용)

#### 인자
- 없음 (self만)

#### 호출되는 곳
```python
# monitoring/logger.py
logger.info(f"설정: {config.to_dict()}")

# database/trades.py
db.save_config_history(config.to_dict())
```

#### 반환값
- Dict[str, Any]: 모든 설정값

#### 구현 코드
```python
def to_dict(self) -> Dict:
    """설정을 딕셔너리로 변환"""
    return {
        'INVESTMENT_AMOUNT': self.INVESTMENT_AMOUNT,
        'POSITION_ALLOCATION': self.POSITION_ALLOCATION,
        'TAKE_PROFIT': self.TAKE_PROFIT,
        'STOP_LOSS': self.STOP_LOSS,
        'TRAILING_ACTIVATION': self.TRAILING_ACTIVATION,
        'TRAILING_STOP': self.TRAILING_STOP,
        'DAILY_LOSS_LIMIT': self.DAILY_LOSS_LIMIT,
        'MONTHLY_DD_LIMIT': self.MONTHLY_DD_LIMIT,
        'CONSECUTIVE_LOSS_LIMIT': self.CONSECUTIVE_LOSS_LIMIT,
        'MIN_AI_CONFIDENCE': self.MIN_AI_CONFIDENCE,
        'AI_CHECK_INTERVAL': self.AI_CHECK_INTERVAL,
        'MAX_HOLDING_TIME': self.MAX_HOLDING_TIME,
        'ORDER_TIMEOUT': self.ORDER_TIMEOUT,
        'LOOP_INTERVAL': self.LOOP_INTERVAL,
        'MODE': self.MODE
    }
```

---

## 📁 core/api_keys.py

### 파일 전체 구조
```python
import os
from typing import Dict, Optional
from dotenv import load_dotenv

class APIKeys:
    REQUIRED_KEYS = {...}
    OPTIONAL_KEYS = {...}
    
    def __init__(self): ...
    def _load_keys(self): ...
    
    @classmethod
    def validate(cls) -> bool: ...
    
    @classmethod
    def get_bybit_keys(cls) -> Dict: ...
    
    @classmethod
    def get_claude_config(cls) -> Dict: ...
```

---

### 📌 클래스 변수

```python
# 필수 키
REQUIRED_KEYS = {
    'BYBIT_API_KEY': 'Bybit 공개 키',
    'BYBIT_API_SECRET': 'Bybit 비밀 키',
    'CLAUDE_API_KEY': 'Claude API 키'
}

# 선택 키 (기본값 포함)
OPTIONAL_KEYS = {
    'BYBIT_TESTNET': ('False', 'Bybit 테스트넷'),
    'CLAUDE_MODEL': ('claude-3-sonnet-20240229', 'Claude 모델'),
    'CLAUDE_MAX_TOKENS': ('1024', '응답 길이'),
    'CLAUDE_TEMPERATURE': ('0.3', '일관성')
}
```

---

### 📌 함수: APIKeys.validate()

```python
@classmethod
def validate(cls) -> bool:
```

#### 역할
API 키 존재 여부 검증

#### 호출되는 곳
```python
# run_paper.py
try:
    APIKeys.validate()
    print("✅ API 키 검증 완료")
except ValueError as e:
    print(f"❌ {e}")
    exit(1)
```

#### 로직
1. cls()로 인스턴스 생성 시도
2. __init__() → _load_keys() 실행
3. 필수 키 누락 시 ValueError

#### 반환값
- bool: True (성공)
- 예외: ValueError (키 누락)

---

### 📌 함수: APIKeys.get_bybit_keys()

```python
@classmethod
def get_bybit_keys(cls) -> Dict:
```

#### 역할
Bybit 관련 키만 추출

#### 호출되는 곳
```python
# exchanges/bybit_live.py
import ccxt

keys = APIKeys.get_bybit_keys()
self.exchange = ccxt.bybit({
    'apiKey': keys['api_key'],
    'secret': keys['api_secret']
})
```

#### 반환값
```python
{
    'api_key': str,
    'api_secret': str,
    'testnet': bool
}
```

---

### 📌 함수: APIKeys.get_claude_config()

```python
@classmethod
def get_claude_config(cls) -> Dict:
```

#### 역할
Claude API 설정 추출

#### 호출되는 곳
```python
# ai/claude_client.py
import anthropic

config = APIKeys.get_claude_config()
self.client = anthropic.Anthropic(
    api_key=config['api_key']
)
self.model = config['model']
```

#### 반환값
```python
{
    'api_key': str,
    'model': str,
    'max_tokens': int,
    'temperature': float
}
```

---

## 📁 core/constants.py

### 주요 상수

```python
# 거래 심볼
SYMBOLS = ['DOGE/USDT', 'SOL/USDT']

# 타임프레임
INDICATOR_TIMEFRAME = '5m'
CANDLE_LIMIT = 200

# 환율 및 수수료
KRW_USD_RATE = 1300
BYBIT_SPOT_FEE = 0.001

# 지표 파라미터
INDICATOR_PARAMS = {
    'RSI': {'period': 14, 'overbought': 70, 'oversold': 30},
    'MACD': {'fast': 12, 'slow': 26, 'signal': 9},
    'BOLLINGER': {'period': 20, 'std': 2.0}
}

# 시스템
CACHE_TTL = 60
DB_PATH = 'storage/trades.db'
```

---

### 📌 함수: get_base_currency(symbol)

```python
def get_base_currency(symbol: str) -> str:
    """'DOGE/USDT' -> 'DOGE'"""
    return symbol.split('/')[0]
```

---

### 📌 함수: get_quote_currency(symbol)

```python
def get_quote_currency(symbol: str) -> str:
    """'DOGE/USDT' -> 'USDT'"""
    return symbol.split('/')[1]
```

---

### 📌 함수: symbol_to_filename(symbol)

```python
def symbol_to_filename(symbol: str) -> str:
    """'DOGE/USDT' -> 'DOGE_USDT'"""
    return symbol.replace('/', '_')
```

---

## 📁 core/exceptions.py

### 예외 계층

```
TradingBotException (기본)
├── APIKeyError
├── APIConnectionError
├── APIRateLimitError
├── InsufficientBalanceError
├── OrderFailedError
├── OrderTimeoutError
├── NetworkError
├── DataFetchError
├── DataValidationError
├── InsufficientDataError
├── AIResponseError
├── AIParseError
├── DailyLossLimitError
├── MonthlyDrawdownError
├── ConsecutiveLossError
└── EmergencyStopError
```

---

### 사용 예시

```python
# 발생
try:
    order = exchange.create_order(...)
except ccxt.InsufficientFunds:
    raise InsufficientBalanceError("잔고 부족")

# 처리
try:
    await process_trading()
except DataFetchError as e:
    logger.warning(f"데이터 오류: {e}")
except DailyLossLimitError as e:
    logger.critical(f"일일 한도: {e}")
    emergency_stop()
```

---

## 전체 의존성 그래프

```
core/ (모두 독립)
├── config.py → 모든 모듈에서 사용
├── api_keys.py → run_*.py, exchanges, ai
├── constants.py → 모든 모듈에서 사용
└── exceptions.py → 모든 모듈에서 사용

의존성 방향: core → 다른 모듈들
(core 내부는 상호 의존 없음)
```

---

## 개발 체크리스트

### config.py
- [ ] @dataclass 사용
- [ ] 모든 속성에 타입 힌트
- [ ] __post_init__() 구현
- [ ] load(mode) 클래스 메서드
- [ ] get_position_size() 구현
- [ ] to_dict() 구현

### api_keys.py
- [ ] REQUIRED_KEYS 정의
- [ ] OPTIONAL_KEYS 정의
- [ ] _load_keys() 구현
- [ ] validate() 구현
- [ ] get_bybit_keys() 구현
- [ ] get_claude_config() 구현

### constants.py
- [ ] 모든 상수 정의
- [ ] 타입 힌트 추가
- [ ] 유틸리티 함수 3개 구현

### exceptions.py
- [ ] TradingBotException 기본 클래스
- [ ] 모든 예외 클래스 정의 (15개+)
- [ ] 계층 구조 확인

---

## .env 파일 예시

```bash
# .env.example
BYBIT_API_KEY=your_api_key
BYBIT_API_SECRET=your_secret
BYBIT_TESTNET=false

CLAUDE_API_KEY=sk-ant-xxxxx
CLAUDE_MODEL=claude-3-sonnet-20240229
CLAUDE_MAX_TOKENS=1024
CLAUDE_TEMPERATURE=0.3
```

---

## 최종 검증

✅ 이 명세서만으로:
1. Config 클래스 동일하게 작성 가능
2. APIKeys 클래스 동일하게 작성 가능
3. 모든 상수 동일하게 정의 가능
4. 모든 예외 동일하게 정의 가능
5. 함수 호출 관계 정확히 파악 가능

---

**문서 버전**: v2.0  
**작성일**: 2025-01-15  
**검증**: ✅ 완료
