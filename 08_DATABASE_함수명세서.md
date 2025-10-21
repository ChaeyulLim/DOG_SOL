# 08_DATABASE 모듈 완벽 함수 명세서

> **목표**: 이 문서만으로 누구나 동일한 코드를 작성할 수 있다

---

## 📋 목차
1. [database/models.py](#databasemodelspy)
2. [database/trades.py](#databasetradespy)
3. [database/learning.py](#databaselearningpy)
4. [전체 의존성 그래프](#전체-의존성-그래프)

---

## 📁 database/models.py

### 파일 전체 구조
```python
import sqlite3
from pathlib import Path
from core.constants import DB_PATH

def init_database() -> None: ...
def get_connection() -> sqlite3.Connection: ...
```

---

### 📌 함수: init_database()

```python
def init_database() -> None:
```

#### 역할
데이터베이스 및 테이블 생성

#### 호출되는 곳
```python
# run_paper.py, run_live.py 시작 시
from database import init_database

init_database()
```

#### 테이블 구조
```sql
-- 1. trades 테이블
CREATE TABLE IF NOT EXISTS trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    
    -- 기본 정보
    timestamp DATETIME NOT NULL,
    symbol TEXT NOT NULL,
    mode TEXT NOT NULL,  -- 'paper', 'live'
    
    -- 거래 정보
    entry_price REAL NOT NULL,
    exit_price REAL,
    quantity REAL NOT NULL,
    pnl_percent REAL,
    pnl_krw REAL,
    
    -- 진입 시점 지표
    rsi_entry REAL,
    macd_entry TEXT,  -- JSON
    bb_entry TEXT,    -- JSON
    volume_ratio REAL,
    
    -- AI 정보
    ai_confidence REAL,
    ai_reasoning TEXT,
    
    -- 청산 정보
    exit_reason TEXT,
    exit_timestamp DATETIME,
    holding_minutes INTEGER,
    
    -- 수수료
    entry_fee REAL,
    exit_fee REAL,
    
    -- 상태
    status TEXT DEFAULT 'OPEN'  -- 'OPEN', 'CLOSED'
);

CREATE INDEX IF NOT EXISTS idx_timestamp ON trades(timestamp);
CREATE INDEX IF NOT EXISTS idx_symbol ON trades(symbol);
CREATE INDEX IF NOT EXISTS idx_status ON trades(status);
```

#### 구현 코드
```python
def init_database() -> None:
    """데이터베이스 초기화"""
    
    db_path = Path(DB_PATH)
    db_path.parent.mkdir(parents=True, exist_ok=True)
    
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    
    # trades 테이블
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS trades (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp DATETIME NOT NULL,
            symbol TEXT NOT NULL,
            mode TEXT NOT NULL,
            entry_price REAL NOT NULL,
            exit_price REAL,
            quantity REAL NOT NULL,
            pnl_percent REAL,
            pnl_krw REAL,
            rsi_entry REAL,
            macd_entry TEXT,
            bb_entry TEXT,
            volume_ratio REAL,
            ai_confidence REAL,
            ai_reasoning TEXT,
            exit_reason TEXT,
            exit_timestamp DATETIME,
            holding_minutes INTEGER,
            entry_fee REAL,
            exit_fee REAL,
            status TEXT DEFAULT 'OPEN'
        )
    """)
    
    cursor.execute("""
        CREATE INDEX IF NOT EXISTS idx_timestamp 
        ON trades(timestamp)
    """)
    
    cursor.execute("""
        CREATE INDEX IF NOT EXISTS idx_symbol 
        ON trades(symbol)
    """)
    
    cursor.execute("""
        CREATE INDEX IF NOT EXISTS idx_status 
        ON trades(status)
    """)
    
    # learning_data 테이블
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS learning_data (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            trade_id INTEGER NOT NULL,
            timestamp DATETIME NOT NULL,
            entry_features TEXT NOT NULL,
            exit_features TEXT,
            market_conditions TEXT,
            outcome TEXT NOT NULL,
            patterns_detected TEXT,
            FOREIGN KEY (trade_id) REFERENCES trades(id)
        )
    """)
    
    # risk_events 테이블
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS risk_events (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp DATETIME NOT NULL,
            event_type TEXT NOT NULL,
            details TEXT,
            action_taken TEXT
        )
    """)
    
    conn.commit()
    conn.close()
    
    print("✅ 데이터베이스 초기화 완료")
```

---

### 📌 함수: get_connection()

```python
def get_connection() -> sqlite3.Connection:
```

#### 역할
데이터베이스 연결 객체 반환

#### 반환값
- `sqlite3.Connection`

#### 구현 코드
```python
def get_connection() -> sqlite3.Connection:
    """DB 연결"""
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row  # Dict 형태로 조회
    return conn
```

---

## 📁 database/trades.py

### 파일 전체 구조
```python
import json
from datetime import datetime
from typing import Dict, List, Optional
from .models import get_connection

class TradeDatabase:
    def save_trade_entry(self, trade_data: Dict) -> int: ...
    
    def update_trade_exit(
        self,
        trade_id: int,
        exit_data: Dict
    ) -> None: ...
    
    def get_open_positions(self) -> List[Dict]: ...
    
    def get_trades_last_n_days(self, days: int = 7) -> List[Dict]: ...
    
    def get_trade_statistics(
        self,
        start_date: Optional[str] = None,
        end_date: Optional[str] = None
    ) -> Dict: ...
```

---

### 📌 클래스: TradeDatabase

#### 목적
거래 데이터 CRUD

---

### 📌 함수: TradeDatabase.save_trade_entry(trade_data)

```python
def save_trade_entry(self, trade_data: Dict) -> int:
```

#### 역할
진입 시 거래 기록 저장

#### 인자
```python
trade_data: Dict = {
    'timestamp': datetime,
    'symbol': str,
    'mode': str,
    'entry_price': float,
    'quantity': float,
    'rsi_entry': float,
    'macd_entry': dict,  # JSON으로 변환됨
    'bb_entry': dict,
    'volume_ratio': float,
    'ai_confidence': float,
    'ai_reasoning': str,
    'entry_fee': float
}
```

#### 호출되는 곳
```python
# engine/base_engine.py - 주문 체결 후
trade_id = self.db.save_trade_entry({
    'timestamp': datetime.now(),
    'symbol': 'DOGE/USDT',
    'mode': 'paper',
    'entry_price': 0.3821,
    'quantity': 1006.0,
    ...
})
```

#### 반환값
- `int`: trade_id (새로 생성된 거래 ID)

#### 구현 코드
```python
def save_trade_entry(self, trade_data: Dict) -> int:
    """
    진입 기록 저장
    
    Returns:
        trade_id: 새로 생성된 거래 ID
    
    Example:
        >>> db = TradeDatabase()
        >>> trade_id = db.save_trade_entry({...})
        >>> print(trade_id)
        123
    """
    conn = get_connection()
    cursor = conn.cursor()
    
    # JSON 변환
    macd_json = json.dumps(trade_data.get('macd_entry', {}))
    bb_json = json.dumps(trade_data.get('bb_entry', {}))
    
    cursor.execute("""
        INSERT INTO trades (
            timestamp, symbol, mode,
            entry_price, quantity,
            rsi_entry, macd_entry, bb_entry, volume_ratio,
            ai_confidence, ai_reasoning,
            entry_fee, status
        ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
    """, (
        trade_data['timestamp'],
        trade_data['symbol'],
        trade_data['mode'],
        trade_data['entry_price'],
        trade_data['quantity'],
        trade_data.get('rsi_entry'),
        macd_json,
        bb_json,
        trade_data.get('volume_ratio'),
        trade_data.get('ai_confidence'),
        trade_data.get('ai_reasoning'),
        trade_data.get('entry_fee', 0),
        'OPEN'
    ))
    
    trade_id = cursor.lastrowid
    
    conn.commit()
    conn.close()
    
    return trade_id
```

---

### 📌 함수: TradeDatabase.update_trade_exit(trade_id, exit_data)

```python
def update_trade_exit(
    self,
    trade_id: int,
    exit_data: Dict
) -> None:
```

#### 역할
청산 시 거래 정보 업데이트

#### 인자
```python
exit_data: Dict = {
    'exit_price': float,
    'pnl_percent': float,
    'pnl_krw': float,
    'exit_reason': str,
    'exit_timestamp': datetime,
    'holding_minutes': int,
    'exit_fee': float
}
```

#### 호출되는 곳
```python
# engine/base_engine.py - 청산 후
self.db.update_trade_exit(trade_id, {
    'exit_price': 0.3895,
    'pnl_percent': 0.0194,
    'pnl_krw': 9700,
    'exit_reason': 'TRAILING_STOP',
    ...
})
```

#### 구현 코드
```python
def update_trade_exit(
    self,
    trade_id: int,
    exit_data: Dict
) -> None:
    """
    청산 정보 업데이트
    
    Example:
        >>> db.update_trade_exit(123, {
        ...     'exit_price': 0.3895,
        ...     'pnl_percent': 0.0194,
        ...     ...
        ... })
    """
    conn = get_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        UPDATE trades SET
            exit_price = ?,
            pnl_percent = ?,
            pnl_krw = ?,
            exit_reason = ?,
            exit_timestamp = ?,
            holding_minutes = ?,
            exit_fee = ?,
            status = 'CLOSED'
        WHERE id = ?
    """, (
        exit_data['exit_price'],
        exit_data['pnl_percent'],
        exit_data['pnl_krw'],
        exit_data['exit_reason'],
        exit_data['exit_timestamp'],
        exit_data['holding_minutes'],
        exit_data.get('exit_fee', 0),
        trade_id
    ))
    
    conn.commit()
    conn.close()
```

---

### 📌 함수: TradeDatabase.get_open_positions()

```python
def get_open_positions(self) -> List[Dict]:
```

#### 역할
현재 열린 포지션 조회

#### 호출되는 곳
```python
# engine/base_engine.py main_loop()
open_positions = self.db.get_open_positions()

for position in open_positions:
    # 청산 체크
```

#### 반환값
```python
List[Dict]:
    [
        {
            'id': 123,
            'symbol': 'DOGE/USDT',
            'entry_price': 0.3821,
            'quantity': 1006.0,
            'timestamp': '2025-01-15 14:30:00',
            ...
        },
        ...
    ]
```

#### 구현 코드
```python
def get_open_positions(self) -> List[Dict]:
    """
    열린 포지션 조회
    
    Returns:
        열린 포지션 리스트
    """
    conn = get_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT * FROM trades
        WHERE status = 'OPEN'
        ORDER BY timestamp DESC
    """)
    
    rows = cursor.fetchall()
    conn.close()
    
    return [dict(row) for row in rows]
```

---

### 📌 함수: TradeDatabase.get_trades_last_n_days(days)

```python
def get_trades_last_n_days(self, days: int = 7) -> List[Dict]:
```

#### 역할
최근 N일간 거래 조회

#### 인자
- `days: int = 7` - 조회 일수

#### 호출되는 곳
```python
# ai/learner.py weekly_analysis()
trades = self.db.get_trades_last_n_days(7)
```

#### 구현 코드
```python
def get_trades_last_n_days(self, days: int = 7) -> List[Dict]:
    """
    최근 N일간 거래
    
    Example:
        >>> trades = db.get_trades_last_n_days(7)
        >>> len(trades)
        15
    """
    conn = get_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT * FROM trades
        WHERE status = 'CLOSED'
        AND timestamp >= datetime('now', '-' || ? || ' days')
        ORDER BY timestamp DESC
    """, (days,))
    
    rows = cursor.fetchall()
    conn.close()
    
    return [dict(row) for row in rows]
```

---

### 📌 함수: TradeDatabase.get_trade_statistics(start_date, end_date)

```python
def get_trade_statistics(
    self,
    start_date: Optional[str] = None,
    end_date: Optional[str] = None
) -> Dict:
```

#### 역할
거래 통계 계산

#### 인자
- `start_date: str` - '2025-01-01' (선택)
- `end_date: str` - '2025-01-31' (선택)

#### 반환값
```python
Dict:
    'total_trades': int = 15
    'win_rate': float = 66.7
    'avg_profit': float = 2.5
    'avg_loss': float = -1.0
    'total_pnl_krw': float = 35420
    'best_trade': Dict = {...}
    'worst_trade': Dict = {...}
```

#### 구현 코드
```python
def get_trade_statistics(
    self,
    start_date: Optional[str] = None,
    end_date: Optional[str] = None
) -> Dict:
    """
    거래 통계
    
    Example:
        >>> stats = db.get_trade_statistics('2025-01-01', '2025-01-31')
        >>> stats['win_rate']
        66.7
    """
    conn = get_connection()
    cursor = conn.cursor()
    
    # 기간 필터
    if start_date and end_date:
        query = """
            SELECT * FROM trades
            WHERE status = 'CLOSED'
            AND date(timestamp) BETWEEN ? AND ?
        """
        cursor.execute(query, (start_date, end_date))
    else:
        query = "SELECT * FROM trades WHERE status = 'CLOSED'"
        cursor.execute(query)
    
    trades = [dict(row) for row in cursor.fetchall()]
    conn.close()
    
    if not trades:
        return {
            'total_trades': 0,
            'win_rate': 0,
            'avg_profit': 0,
            'avg_loss': 0,
            'total_pnl_krw': 0
        }
    
    # 통계 계산
    total = len(trades)
    winners = [t for t in trades if t['pnl_percent'] > 0]
    losers = [t for t in trades if t['pnl_percent'] <= 0]
    
    win_rate = (len(winners) / total) * 100
    
    avg_profit = sum(t['pnl_percent'] for t in winners) / len(winners) * 100 if winners else 0
    avg_loss = sum(t['pnl_percent'] for t in losers) / len(losers) * 100 if losers else 0
    
    total_pnl = sum(t['pnl_krw'] for t in trades)
    
    best = max(trades, key=lambda x: x['pnl_percent'])
    worst = min(trades, key=lambda x: x['pnl_percent'])
    
    return {
        'total_trades': total,
        'win_rate': round(win_rate, 1),
        'avg_profit': round(avg_profit, 2),
        'avg_loss': round(avg_loss, 2),
        'total_pnl_krw': round(total_pnl, 0),
        'best_trade': {
            'symbol': best['symbol'],
            'pnl_percent': best['pnl_percent'] * 100
        },
        'worst_trade': {
            'symbol': worst['symbol'],
            'pnl_percent': worst['pnl_percent'] * 100
        }
    }
```

---

## 📁 database/learning.py

### 파일 전체 구조
```python
import json
from datetime import datetime
from typing import Dict
from .models import get_connection

class LearningDatabase:
    def save_learning_data(
        self,
        trade_id: int,
        learning_data: Dict
    ) -> None: ...
    
    def get_learning_history(self, days: int = 30) -> List[Dict]: ...
```

---

### 📌 클래스: LearningDatabase

#### 목적
AI 학습 데이터 저장

---

### 📌 함수: LearningDatabase.save_learning_data(trade_id, learning_data)

```python
def save_learning_data(
    self,
    trade_id: int,
    learning_data: Dict
) -> None:
```

#### 역할
거래 종료 후 학습 데이터 저장

#### 인자
```python
learning_data: Dict = {
    'entry_features': dict,  # RSI, MACD, 시간대 등
    'exit_features': dict,   # 보유시간, 최고수익 등
    'market_conditions': dict,  # BTC 방향, Fear&Greed
    'outcome': dict,         # 결과 (승/패, 수익률)
    'patterns_detected': list  # 감지된 패턴
}
```

#### 호출되는 곳
```python
# engine/base_engine.py - 청산 후
self.learning_db.save_learning_data(trade_id, {
    'entry_features': {...},
    'outcome': {'result': 'WIN', 'pnl': 0.0194}
})
```

#### 구현 코드
```python
def save_learning_data(
    self,
    trade_id: int,
    learning_data: Dict
) -> None:
    """
    학습 데이터 저장
    """
    conn = get_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        INSERT INTO learning_data (
            trade_id, timestamp,
            entry_features, exit_features,
            market_conditions, outcome,
            patterns_detected
        ) VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (
        trade_id,
        datetime.now(),
        json.dumps(learning_data.get('entry_features', {})),
        json.dumps(learning_data.get('exit_features', {})),
        json.dumps(learning_data.get('market_conditions', {})),
        json.dumps(learning_data.get('outcome', {})),
        json.dumps(learning_data.get('patterns_detected', []))
    ))
    
    conn.commit()
    conn.close()
```

---

### 📌 함수: LearningDatabase.get_learning_history(days)

```python
def get_learning_history(self, days: int = 30) -> List[Dict]:
```

#### 역할
학습 데이터 조회

#### 구현 코드
```python
def get_learning_history(self, days: int = 30) -> List[Dict]:
    """학습 이력 조회"""
    conn = get_connection()
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT * FROM learning_data
        WHERE timestamp >= datetime('now', '-' || ? || ' days')
        ORDER BY timestamp DESC
    """, (days,))
    
    rows = cursor.fetchall()
    conn.close()
    
    # JSON 파싱
    result = []
    for row in rows:
        data = dict(row)
        data['entry_features'] = json.loads(data['entry_features'])
        data['exit_features'] = json.loads(data['exit_features'])
        data['outcome'] = json.loads(data['outcome'])
        result.append(data)
    
    return result
```

---

## 전체 의존성 그래프

### DATABASE 모듈 구조
```
models.py (기본)
├── trades.py → models.get_connection()
└── learning.py → models.get_connection()
```

### 사용하는 모듈
```
core/constants → DB_PATH
sqlite3 (표준 라이브러리)
json (표준 라이브러리)
```

### 사용되는 곳
```
engine/base_engine.py
├── TradeDatabase.save_trade_entry()
├── TradeDatabase.update_trade_exit()
├── TradeDatabase.get_open_positions()
└── LearningDatabase.save_learning_data()

ai/learner.py
└── TradeDatabase.get_trades_last_n_days()
```

---

## 개발 체크리스트

### models.py
- [ ] init_database() - 테이블 생성
- [ ] get_connection() - 연결 반환
- [ ] 인덱스 생성 (timestamp, symbol, status)

### trades.py
- [ ] TradeDatabase 클래스
- [ ] save_trade_entry() - 진입 저장
- [ ] update_trade_exit() - 청산 업데이트
- [ ] get_open_positions() - 열린 포지션
- [ ] get_trades_last_n_days() - 기간 조회
- [ ] get_trade_statistics() - 통계 계산

### learning.py
- [ ] LearningDatabase 클래스
- [ ] save_learning_data() - 학습 데이터 저장
- [ ] get_learning_history() - 학습 이력 조회

---

## 테스트 시나리오

### trades.py 테스트
```python
from database import init_database, TradeDatabase
from datetime import datetime

# 1. 초기화
init_database()
db = TradeDatabase()

# 2. 진입 저장
trade_id = db.save_trade_entry({
    'timestamp': datetime.now(),
    'symbol': 'DOGE/USDT',
    'mode': 'paper',
    'entry_price': 0.3821,
    'quantity': 1006.0,
    'ai_confidence': 0.75,
    'entry_fee': 0.38
})
assert trade_id > 0

# 3. 열린 포지션 조회
positions = db.get_open_positions()
assert len(positions) == 1
assert positions[0]['symbol'] == 'DOGE/USDT'

# 4. 청산 업데이트
db.update_trade_exit(trade_id, {
    'exit_price': 0.3895,
    'pnl_percent': 0.0194,
    'pnl_krw': 9700,
    'exit_reason': 'TRAILING_STOP',
    'exit_timestamp': datetime.now(),
    'holding_minutes': 135,
    'exit_fee': 0.39
})

# 5. 통계 조회
stats = db.get_trade_statistics()
assert stats['total_trades'] == 1
assert stats['win_rate'] == 100.0
```

---

## 주요 특징

### 1. SQLite 사용
- 별도 서버 불필요
- 파일 기반 (storage/trades.db)
- 가볍고 빠름

### 2. JSON 저장
- 지표 정보는 JSON 형태
- 유연한 구조
- 파싱/언파싱 자동

### 3. 인덱스 최적화
- timestamp, symbol, status 인덱스
- 빠른 조회 성능

### 4. 학습 데이터 분리
- trades: 거래 기록
- learning_data: AI 학습용
- 외래키로 연결

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 7 (데이터베이스 레이어)  
**검증**: ✅ 완료