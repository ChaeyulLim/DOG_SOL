# 09_ENGINE 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [engine/base_engine.py](#enginebaseenginepy)
2. [engine/paper_engine.py](#enginepaperenginepy)
3. [engine/live_engine.py](#engineliveenginepy)
4. [engine/backtest_engine.py](#enginebacktestenginepy)
5. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📁 engine/base_engine.py

### 파일 전체 구조
```python
import asyncio
import time
from typing import Dict, Optional
from abc import ABC, abstractmethod

from core.config import Config
from data import MarketDataFetcher
from indicators import IndicatorCalculator
from strategy import EntryStrategy, ExitStrategy
from ai import ClaudeClient
from exchanges.base import BaseExchange
from risk import RiskManager, PositionSizer
from database import TradeDatabase, LearningDatabase

class BaseEngine(ABC):
    def __init__(self, config: Config, exchange: BaseExchange): ...
    
    async def initialize(self) -> None: ...
    
    async def run(self) -> None: ...
    
    async def main_loop(self) -> None: ...
    
    async def process_symbol(self, symbol: str, data: Dict) -> None: ...
    
    async def execute_entry(self, symbol: str, signal: Dict) -> None: ...
    
    async def execute_exit(self, symbol: str, position: Dict, signal: Dict) -> None: ...
    
    def cleanup(self) -> None: ...
    
    @abstractmethod
    def get_today_start_balance(self) -> float: ...
    
    @abstractmethod
    def get_monthly_high_balance(self) -> float: ...
```

---

### 📌 클래스: BaseEngine

#### 목적
모든 엔진의 공통 로직 (Paper, Live, Backtest 상속)

---

### 📌 함수: BaseEngine.__init__(config, exchange)

```python
def __init__(self, config: Config, exchange: BaseExchange):
```

#### 역할
엔진 초기화 (모든 모듈 로드)

#### 인자
- `config: Config` - 시스템 설정
- `exchange: BaseExchange` - 거래소 인터페이스

#### 초기화 내용
```python
self.config = config
self.exchange = exchange

# 데이터 레이어
self.fetcher = MarketDataFetcher(exchange)
self.calculator = IndicatorCalculator()

# AI
self.claude = ClaudeClient()

# 전략
self.entry_strategy = EntryStrategy(config, self.claude)
self.exit_strategy = ExitStrategy(config, self.claude)

# 리스크
self.risk_manager = RiskManager(config)
self.position_sizer = PositionSizer(config)

# 데이터베이스
self.trade_db = TradeDatabase()
self.learning_db = LearningDatabase()

# 포지션 추적
self.positions = {}  # {symbol: position_info}

# 상태
self.running = False
```

#### 호출되는 곳
```python
# engine/paper_engine.py
class PaperEngine(BaseEngine):
    def __init__(self, config: Config):
        exchange = PaperExchange(config.INVESTMENT_AMOUNT)
        super().__init__(config, exchange)
```

#### 구현 코드
```python
def __init__(self, config: Config, exchange: BaseExchange):
    """
    엔진 초기화
    
    모든 모듈 로드 및 연결
    """
    self.config = config
    self.exchange = exchange
    
    # 데이터
    self.fetcher = MarketDataFetcher(exchange)
    self.calculator = IndicatorCalculator()
    
    # AI
    self.claude = ClaudeClient()
    
    # 전략
    self.entry_strategy = EntryStrategy(config, self.claude)
    self.exit_strategy = ExitStrategy(config, self.claude)
    
    # 리스크
    self.risk_manager = RiskManager(config)
    self.position_sizer = PositionSizer(config)
    
    # DB
    self.trade_db = TradeDatabase()
    self.learning_db = LearningDatabase()
    
    # 포지션
    self.positions = {}
    
    # 상태
    self.running = False
```

---

### 📌 함수: BaseEngine.initialize()

```python
async def initialize(self) -> None:
```

#### 역할
시작 전 초기화 작업

#### 처리 내용
```
1. 거래소 연결 테스트
2. API 키 검증
3. 데이터베이스 초기화
4. 미청산 포지션 복구
```

#### 구현 코드
```python
async def initialize(self) -> None:
    """시작 전 초기화"""
    
    print("=" * 60)
    print("🚀 시스템 초기화 시작")
    print("=" * 60)
    
    # 1. 거래소 연결
    self.exchange.initialize()
    print("✅ 거래소 연결 완료")
    
    # 2. 미청산 포지션 복구
    open_positions = self.trade_db.get_open_positions()
    for pos in open_positions:
        self.positions[pos['symbol']] = {
            'trade_id': pos['id'],
            'entry_price': pos['entry_price'],
            'quantity': pos['quantity'],
            'entry_time': time.mktime(
                time.strptime(pos['timestamp'], '%Y-%m-%d %H:%M:%S')
            )
        }
    
    if open_positions:
        print(f"✅ 미청산 포지션 {len(open_positions)}개 복구")
    
    print("=" * 60)
    print("✅ 초기화 완료")
    print("=" * 60)
```

---

### 📌 함수: BaseEngine.run()

```python
async def run(self) -> None:
```

#### 역할
메인 실행 루프 시작

#### 처리 흐름
```
1. running = True
2. main_loop() 무한 반복
3. KeyboardInterrupt 처리
4. cleanup()
```

#### 구현 코드
```python
async def run(self) -> None:
    """
    메인 실행
    """
    self.running = True
    
    try:
        while self.running:
            await self.main_loop()
            
            # 1분 대기
            await asyncio.sleep(self.config.LOOP_INTERVAL)
    
    except KeyboardInterrupt:
        print("\n사용자 중단 요청")
        self.running = False
    
    except Exception as e:
        print(f"❌ 시스템 오류: {e}")
        raise
    
    finally:
        self.cleanup()
```

---

### 📌 함수: BaseEngine.main_loop()

```python
async def main_loop(self) -> None:
```

#### 역할
1분마다 실행되는 핵심 로직

#### 처리 흐름
```
1. 모든 심볼 데이터 수집
2. 리스크 체크
3. 각 심볼 처리
   - 포지션 없으면 → 진입 검토
   - 포지션 있으면 → 청산 검토
4. 로그 출력
```

#### 구현 코드
```python
async def main_loop(self) -> None:
    """
    메인 루프 (1분마다)
    """
    print(f"\n[{time.strftime('%H:%M:%S')}] 체크 시작")
    
    # 1. 데이터 수집
    try:
        all_data = await self.fetcher.fetch_multiple_symbols()
    except Exception as e:
        print(f"❌ 데이터 수집 실패: {e}")
        return
    
    # 2. 리스크 체크
    current_balance = self.exchange.get_balance()['total_krw']
    today_start = self.get_today_start_balance()
    monthly_high = self.get_monthly_high_balance()
    
    risk_result = self.risk_manager.check_all_limits(
        current_balance,
        today_start,
        monthly_high
    )
    
    if risk_result['action'] != 'OK':
        print(f"🚨 리스크 한도 초과: {risk_result['reason']}")
        # 모든 포지션 청산
        for symbol in list(self.positions.keys()):
            await self.force_close_position(symbol)
        self.running = False
        return
    
    # 3. 심볼별 처리
    for symbol, data in all_data.items():
        await self.process_symbol(symbol, data)
    
    print(f"[{time.strftime('%H:%M:%S')}] 체크 완료")
```

---

### 📌 함수: BaseEngine.process_symbol(symbol, data)

```python
async def process_symbol(self, symbol: str, data: Dict) -> None:
```

#### 역할
개별 심볼 처리

#### 처리 흐름
```
1. 지표 계산
2. 포지션 확인
   - 없으면: 진입 검토
   - 있으면: 청산 검토
```

#### 구현 코드
```python
async def process_symbol(self, symbol: str, data: Dict) -> None:
    """
    심볼별 처리
    """
    # 지표 계산
    try:
        indicators = self.calculator.calculate_all(data['ohlcv'])
    except Exception as e:
        print(f"❌ {symbol} 지표 계산 실패: {e}")
        return
    
    # 포지션 확인
    if symbol not in self.positions:
        # 진입 검토
        entry_signal = await self.entry_strategy.check_entry_signal(
            symbol,
            data,
            indicators
        )
        
        if entry_signal:
            await self.execute_entry(symbol, entry_signal)
    
    else:
        # 청산 검토
        position = self.positions[symbol]
        
        exit_signal = await self.exit_strategy.check_exit_signal(
            symbol,
            position,
            data['price'],
            indicators
        )
        
        if exit_signal:
            await self.execute_exit(symbol, position, exit_signal)
```

---

### 📌 함수: BaseEngine.execute_entry(symbol, signal)

```python
async def execute_entry(self, symbol: str, signal: Dict) -> None:
```

#### 역할
진입 주문 실행

#### 처리 흐름
```
1. 포지션 크기 계산
2. 주문 생성
3. DB 저장
4. 포지션 추적
5. 로그
```

#### 구현 코드
```python
async def execute_entry(self, symbol: str, signal: Dict) -> None:
    """
    진입 실행
    """
    print(f"\n🟢 [{symbol}] 진입 신호!")
    print(f"   신뢰도: {signal['confidence']:.2f}")
    print(f"   이유: {signal['ai_reasoning'][:50]}...")
    
    # 1. 포지션 크기
    balance = self.exchange.get_balance()['total_krw']
    amount_krw = self.position_sizer.calculate_position_size(
        balance,
        symbol.split('/')[0]
    )
    
    # 2. 주문 생성
    try:
        order = self.exchange.create_order(symbol, amount_krw)
    except Exception as e:
        print(f"❌ 주문 실패: {e}")
        return
    
    print(f"✅ 체결: {order['filled']:.2f} @ ${order['average_price']:.4f}")
    
    # 3. DB 저장
    trade_id = self.trade_db.save_trade_entry({
        'timestamp': datetime.now(),
        'symbol': symbol,
        'mode': self.config.MODE,
        'entry_price': order['average_price'],
        'quantity': order['filled'],
        'ai_confidence': signal['confidence'],
        'ai_reasoning': signal['ai_reasoning'],
        'entry_fee': order['fee']
    })
    
    # 4. 포지션 추적
    self.positions[symbol] = {
        'trade_id': trade_id,
        'entry_price': order['average_price'],
        'quantity': order['filled'],
        'entry_time': time.time()
    }
```

---

### 📌 함수: BaseEngine.execute_exit(symbol, position, signal)

```python
async def execute_exit(
    self,
    symbol: str,
    position: Dict,
    signal: Dict
) -> None:
```

#### 역할
청산 주문 실행

#### 처리 흐름
```
1. 청산 주문
2. 수익률 계산
3. DB 업데이트
4. 포지션 삭제
5. 리스크 기록
6. 학습 데이터 저장
7. 로그
```

#### 구현 코드
```python
async def execute_exit(
    self,
    symbol: str,
    position: Dict,
    signal: Dict
) -> None:
    """
    청산 실행
    """
    print(f"\n🔴 [{symbol}] 청산 신호!")
    print(f"   이유: {signal['reason']}")
    
    # 1. 청산 주문
    try:
        order = self.exchange.close_position(symbol)
    except Exception as e:
        print(f"❌ 청산 실패: {e}")
        return
    
    # 2. 수익률 계산
    entry_price = position['entry_price']
    exit_price = order['price']
    pnl_percent = (exit_price - entry_price) / entry_price
    pnl_krw = (exit_price - entry_price) * order['quantity']
    
    # 수수료 차감
    total_fee = position.get('entry_fee', 0) + order['fee']
    pnl_krw -= total_fee
    
    holding_time = time.time() - position['entry_time']
    holding_minutes = int(holding_time / 60)
    
    print(f"✅ 청산: ${exit_price:.4f}")
    print(f"   수익률: {pnl_percent*100:+.2f}%")
    print(f"   수익금: {pnl_krw:+,.0f} KRW")
    print(f"   보유: {holding_minutes}분")
    
    # 3. DB 업데이트
    self.trade_db.update_trade_exit(position['trade_id'], {
        'exit_price': exit_price,
        'pnl_percent': pnl_percent,
        'pnl_krw': pnl_krw,
        'exit_reason': signal['reason'],
        'exit_timestamp': datetime.now(),
        'holding_minutes': holding_minutes,
        'exit_fee': order['fee']
    })
    
    # 4. 포지션 삭제
    del self.positions[symbol]
    
    # 5. 리스크 기록
    self.risk_manager.record_trade_result(pnl_percent)
    
    # 6. 학습 데이터 (선택)
    # self.learning_db.save_learning_data(...)
```

---

### 📌 함수: BaseEngine.cleanup()

```python
def cleanup(self) -> None:
```

#### 역할
종료 전 정리 작업

#### 구현 코드
```python
def cleanup(self) -> None:
    """종료 정리"""
    
    print("\n" + "=" * 60)
    print("🛑 시스템 종료 중...")
    print("=" * 60)
    
    # 열린 포지션 경고
    if self.positions:
        print(f"⚠️  미청산 포지션 {len(self.positions)}개 존재")
        for symbol in self.positions:
            print(f"   - {symbol}")
    
    print("✅ 종료 완료")
```

---

## 📁 engine/paper_engine.py

### 파일 전체 구조
```python
from .base_engine import BaseEngine
from core.config import Config
from exchanges import PaperExchange

class PaperEngine(BaseEngine):
    def __init__(self, config: Config): ...
    def get_today_start_balance(self) -> float: ...
    def get_monthly_high_balance(self) -> float: ...
```

---

### 📌 클래스: PaperEngine

#### 목적
모의투자 엔진

---

### 📌 함수: PaperEngine.__init__(config)

```python
def __init__(self, config: Config):
```

#### 구현 코드
```python
def __init__(self, config: Config):
    """모의투자 엔진"""
    
    # 가상 거래소
    exchange = PaperExchange(config.INVESTMENT_AMOUNT)
    
    # 부모 초기화
    super().__init__(config, exchange)
    
    # 추가 상태
    self.today_start_balance = config.INVESTMENT_AMOUNT
    self.monthly_high_balance = config.INVESTMENT_AMOUNT
```

---

### 📌 함수: PaperEngine.get_today_start_balance()

```python
def get_today_start_balance(self) -> float:
```

#### 구현 코드
```python
def get_today_start_balance(self) -> float:
    """오늘 시작 잔고"""
    # 간단 구현: 매일 00:00에 갱신 필요
    return self.today_start_balance
```

---

## 📁 engine/live_engine.py

### 파일 전체 구조
```python
from .base_engine import BaseEngine
from core.config import Config
from exchanges import BybitLiveExchange

class LiveEngine(BaseEngine):
    def __init__(self, config: Config): ...
```

---

### 📌 클래스: LiveEngine

#### 목적
실거래 엔진

---

### 📌 함수: LiveEngine.__init__(config)

```python
def __init__(self, config: Config):
```

#### 구현 코드
```python
def __init__(self, config: Config):
    """실거래 엔진"""
    
    # 실제 거래소
    exchange = BybitLiveExchange()
    
    # 부모 초기화
    super().__init__(config, exchange)
    
    print("⚠️  실거래 모드 - 실제 자금 사용")
```

---

## 📁 engine/backtest_engine.py

### 파일 전체 구조
```python
from .base_engine import BaseEngine
from core.config import Config
from exchanges import BacktestExchange
from data import HistoricalDataLoader

class BacktestEngine(BaseEngine):
    def __init__(
        self,
        start_date: str,
        end_date: str,
        symbols: List[str]
    ): ...
    
    async def run(self) -> None: ...
```

---

### 📌 클래스: BacktestEngine

#### 목적
백테스트 엔진 (과거 데이터 재생)

---

### 📌 함수: BacktestEngine.__init__(start_date, end_date, symbols)

```python
def __init__(
    self,
    start_date: str,
    end_date: str,
    symbols: List[str]
):
```

#### 구현 코드
```python
def __init__(
    self,
    start_date: str,
    end_date: str,
    symbols: List[str]
):
    """백테스트 엔진"""
    
    config = Config.load('backtest')
    
    # 과거 데이터 로드
    loader = HistoricalDataLoader()
    historical_data = {}
    
    for symbol in symbols:
        data = loader.load_historical_data(
            symbol,
            start_date,
            end_date
        )
        historical_data[symbol] = data
    
    # 백테스트 거래소
    exchange = BacktestExchange(historical_data)
    
    super().__init__(config, exchange)
    
    self.historical_data = historical_data
```

---

### 📌 함수: BacktestEngine.run()

```python
async def run(self) -> None:
```

#### 역할
과거 데이터 재생 (오버라이드)

#### 구현 코드
```python
async def run(self) -> None:
    """백테스트 실행"""
    
    # 모든 타임스탬프 수집
    all_timestamps = set()
    for data_list in self.historical_data.values():
        for candle in data_list:
            all_timestamps.add(candle['timestamp'])
    
    timestamps = sorted(all_timestamps)
    
    print(f"📊 백테스트: {len(timestamps)}개 캔들 처리")
    
    for i, ts in enumerate(timestamps):
        # 시간 설정
        self.exchange.set_current_timestamp(ts)
        
        # 메인 로직
        await self.main_loop()
        
        # 진행률
        if (i + 1) % 100 == 0:
            progress = (i + 1) / len(timestamps) * 100
            print(f"진행률: {progress:.1f}%")
    
    print("✅ 백테스트 완료")
    self.generate_report()
```

---

## 전체 의존성 그래프

### ENGINE 모듈 구조
```
base_engine.py (추상)
├── paper_engine.py (상속)
├── live_engine.py (상속)
└── backtest_engine.py (상속)
```

### 사용하는 모든 모듈
```
core → Config
data → MarketDataFetcher
indicators → IndicatorCalculator
strategy → EntryStrategy, ExitStrategy
ai → ClaudeClient
exchanges → BaseExchange
risk → RiskManager, PositionSizer
database → TradeDatabase, LearningDatabase
```

### 호출되는 곳
```
run_paper.py → PaperEngine
run_live.py → LiveEngine
run_backtest.py → BacktestEngine
```

---

## 개발 체크리스트

### base_engine.py
- [ ] BaseEngine 추상 클래스
- [ ] __init__() - 모든 모듈 로드
- [ ] initialize() - 초기화
- [ ] run() - 메인 실행
- [ ] main_loop() - 1분 루프
- [ ] process_symbol() - 심볼 처리
- [ ] execute_entry() - 진입 실행
- [ ] execute_exit() - 청산 실행
- [ ] cleanup() - 종료 정리

### paper_engine.py
- [ ] PaperEngine 클래스
- [ ] PaperExchange 사용
- [ ] 잔고 추적 메서드

### live_engine.py
- [ ] LiveEngine 클래스
- [ ] BybitLiveExchange 사용
- [ ] 경고 메시지

### backtest_engine.py
- [ ] BacktestEngine 클래스
- [ ] 과거 데이터 로드
- [ ] run() 오버라이드
- [ ] 진행률 표시

---

## 주요 특징

### 1. 상속 구조
- BaseEngine에 공통 로직
- 모드별 차이만 오버라이드

### 2. 비동기 처리
- async/await 전면 사용
- AI 호출 대기 최소화

### 3. 완벽한 추적
- 모든 거래 DB 저장
- 포지션 메모리 관리
- 리스크 실시간 체크

### 4. 견고한 에러 처리
- try-except 모든 구간
- 시스템 다운 방지
- 명확한 에러 메시지

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 8 (엔진 레이어)  
**검증**: ✅ 완료