
<artifact identifier="storage-structure-doc" type="text/markdown" title="12_STORAGE_구조설명.md">
# 12_STORAGE 디렉토리 구조 설명

> **목적**: 데이터 저장소 구조 및 관리 방법

---

## 📋 목차
1. [디렉토리 구조](#디렉토리-구조)
2. [historical/ - 과거 데이터](#historical---과거-데이터)
3. [cache/ - 임시 캐시](#cache---임시-캐시)
4. [trades.db - 거래 데이터베이스](#tradesdb---거래-데이터베이스)
5. [관리 및 유지보수](#관리-및-유지보수)

---

## 디렉토리 구조

```
storage/
├── historical/              # 백테스트용 과거 데이터
│   ├── DOGE_USDT_5m.csv
│   ├── SOL_USDT_5m.csv
│   ├── DOGE_USDT_1h.csv
│   ├── SOL_USDT_1h.csv
│   └── README.md
│
├── cache/                   # 1분간 임시 캐시
│   ├── market_data.json
│   ├── indicators.json
│   └── .gitkeep
│
├── system_state.json        # 시스템 상태 백업 (5분마다)
├── trades.db               # SQLite 거래 DB
├── trades.db-journal       # SQLite 저널 (자동 생성)
└── README.md
```

---

## historical/ - 과거 데이터

### 목적
백테스트용 과거 OHLCV 데이터 저장

### CSV 파일 형식

**파일명 규칙**:
```
{SYMBOL}_{TIMEFRAME}.csv
예: DOGE_USDT_5m.csv, SOL_USDT_1h.csv
```

**CSV 구조**:
```csv
timestamp,open,high,low,close,volume
1640000000,0.3821,0.3895,0.3800,0.3850,1234567890
1640000300,0.3850,0.3920,0.3840,0.3900,1345678901
...
```

**컬럼 설명**:
- `timestamp`: Unix 타임스탬프 (초)
- `open`: 시가 (USDT)
- `high`: 고가 (USDT)
- `low`: 저가 (USDT)
- `close`: 종가 (USDT)
- `volume`: 거래량 (코인 수량)

### 데이터 수집 방법

**옵션 1: Bybit API로 직접 수집**
```python
# scripts/download_historical_data.py
import ccxt
import pandas as pd
from datetime import datetime, timedelta

def download_historical_data(symbol, timeframe, days):
    exchange = ccxt.bybit()
    
    since = exchange.parse8601(
        (datetime.now() - timedelta(days=days)).isoformat()
    )
    
    all_ohlcv = []
    
    while True:
        ohlcv = exchange.fetch_ohlcv(
            symbol, 
            timeframe, 
            since=since,
            limit=1000
        )
        
        if not ohlcv:
            break
        
        all_ohlcv.extend(ohlcv)
        since = ohlcv[-1][0] + 1
        
        if len(ohlcv) < 1000:
            break
    
    # CSV 저장
    df = pd.DataFrame(
        all_ohlcv,
        columns=['timestamp', 'open', 'high', 'low', 'close', 'volume']
    )
    
    # 타임스탬프를 초 단위로 변환
    df['timestamp'] = df['timestamp'] // 1000
    
    filename = f"storage/historical/{symbol.replace('/', '_')}_{timeframe}.csv"
    df.to_csv(filename, index=False)
    
    print(f"✅ {filename} 저장 완료 ({len(df)}개)")

# 사용
download_historical_data('DOGE/USDT', '5m', 180)  # 6개월
download_historical_data('SOL/USDT', '5m', 180)
```

**옵션 2: 외부 데이터 사용**
- CryptoDataDownload: https://www.cryptodatadownload.com/
- Binance Vision: https://data.binance.vision/
- 형식 맞춰서 저장

### 데이터 검증
```python
# scripts/validate_historical_data.py
import pandas as pd

def validate_csv(filepath):
    df = pd.read_csv(filepath)
    
    # 필수 컬럼 확인
    required = ['timestamp', 'open', 'high', 'low', 'close', 'volume']
    assert all(col in df.columns for col in required)
    
    # 데이터 타입 확인
    assert df['timestamp'].dtype in ['int64', 'int32']
    assert all(df[col].dtype == 'float64' for col in ['open', 'high', 'low', 'close', 'volume'])
    
    # 논리 검증
    assert (df['high'] >= df['low']).all()
    assert (df['high'] >= df['open']).all()
    assert (df['high'] >= df['close']).all()
    assert (df['low'] <= df['open']).all()
    assert (df['low'] <= df['close']).all()
    
    # 시간 순서 확인
    assert df['timestamp'].is_monotonic_increasing
    
    print(f"✅ {filepath} 검증 완료")
    print(f"   데이터 개수: {len(df)}")
    print(f"   기간: {df['timestamp'].min()} ~ {df['timestamp'].max()}")
    
validate_csv('storage/historical/DOGE_USDT_5m.csv')
```

### README.md (historical/)
```markdown
# Historical Data

백테스트용 과거 OHLCV 데이터

## 파일 목록
- `DOGE_USDT_5m.csv`: DOGE 5분봉 (6개월)
- `SOL_USDT_5m.csv`: SOL 5분봉 (6개월)

## 데이터 갱신
```bash
python scripts/download_historical_data.py
```

## 주의사항
- 용량이 크므로 git에 커밋하지 않음 (.gitignore 포함)
- 최소 6개월 이상 데이터 권장
```

---

## cache/ - 임시 캐시

### 목적
1분간 유효한 API 응답 캐시

### 캐시 파일 구조

**market_data.json**:
```json
{
  "DOGE/USDT": {
    "price": 0.3821,
    "volume_24h": 1234567890,
    "timestamp": 1640000000,
    "expires_at": 1640000060
  },
  "SOL/USDT": {
    "price": 98.52,
    "volume_24h": 9876543210,
    "timestamp": 1640000000,
    "expires_at": 1640000060
  }
}
```

**indicators.json**:
```json
{
  "DOGE/USDT": {
    "rsi": {"value": 45.2, "oversold": false},
    "macd": {"value": 0.0015, "golden_cross": true},
    "timestamp": 1640000000,
    "expires_at": 1640000060
  }
}
```

### 캐시 관리

**data/cache.py에서 자동 관리**:
```python
# 읽기
cached = cache.get('market_data', 'DOGE/USDT')
if cached and cached['expires_at'] > time.time():
    return cached

# 쓰기
cache.set('market_data', 'DOGE/USDT', {
    'price': 0.3821,
    'timestamp': time.time(),
    'expires_at': time.time() + 60  # 1분 후 만료
})
```

### .gitkeep
```
# 빈 디렉토리를 git에 포함시키기 위한 파일
```

---

## trades.db - 거래 데이터베이스

### 목적
모든 거래 기록 영구 저장 (SQLite)

### 데이터베이스 스키마

**trades 테이블**:
```sql
CREATE TABLE trades (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME NOT NULL,
    symbol TEXT NOT NULL,
    mode TEXT NOT NULL,  -- 'paper', 'live', 'backtest'
    
    -- 거래 정보
    entry_price REAL NOT NULL,
    exit_price REAL,
    quantity REAL NOT NULL,
    pnl_percent REAL,
    pnl_krw REAL,
    
    -- 지표
    rsi_entry REAL,
    macd_entry TEXT,  -- JSON
    bb_entry TEXT,    -- JSON
    volume_ratio REAL,
    
    -- AI
    ai_confidence REAL,
    ai_reasoning TEXT,
    
    -- 청산
    exit_reason TEXT,
    exit_timestamp DATETIME,
    holding_minutes INTEGER,
    
    -- 수수료
    entry_fee REAL,
    exit_fee REAL,
    
    -- 상태
    status TEXT DEFAULT 'OPEN',  -- 'OPEN', 'CLOSED'
    
    INDEX idx_timestamp (timestamp),
    INDEX idx_symbol (symbol),
    INDEX idx_status (status)
);
```

**learning_data 테이블**:
```sql
CREATE TABLE learning_data (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    trade_id INTEGER NOT NULL,
    timestamp DATETIME NOT NULL,
    
    entry_features TEXT NOT NULL,  -- JSON
    exit_features TEXT,            -- JSON
    market_conditions TEXT,        -- JSON
    outcome TEXT NOT NULL,          -- JSON
    patterns_detected TEXT,        -- JSON
    
    FOREIGN KEY (trade_id) REFERENCES trades(id)
);
```

**risk_events 테이블**:
```sql
CREATE TABLE risk_events (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp DATETIME NOT NULL,
    event_type TEXT NOT NULL,
    details TEXT,      -- JSON
    action_taken TEXT
);
```

### 데이터베이스 관리

**초기 생성** (database/models.py):
```python
import sqlite3

def initialize_database():
    conn = sqlite3.connect('storage/trades.db')
    cursor = conn.cursor()
    
    # trades 테이블 생성
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS trades (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp DATETIME NOT NULL,
            ...
        )
    ''')
    
    # 인덱스 생성
    cursor.execute('CREATE INDEX IF NOT EXISTS idx_timestamp ON trades(timestamp)')
    
    conn.commit()
    conn.close()
```

**백업**:
```bash
# 매일 자동 백업
cp storage/trades.db storage/backups/trades_$(date +%Y%m%d).db
```

**용량 관리**:
```python
# 오래된 데이터 정리 (선택)
DELETE FROM trades WHERE timestamp < date('now', '-1 year');
VACUUM;  # DB 파일 크기 최적화
```

---

## system_state.json

### 목적
시스템 비정상 종료 시 복구용 상태 저장

### 저장 내용
```json
{
  "timestamp": 1640000000,
  "mode": "paper",
  "balance": 1035420,
  "positions": {
    "DOGE/USDT": {
      "trade_id": 123,
      "entry_price": 0.3821,
      "quantity": 1006,
      "entry_time": 1640000000
    }
  },
  "risk_state": {
    "daily_loss": 0.012,
    "monthly_dd": -0.021,
    "consecutive_losses": 0
  },
  "config": {
    "INVESTMENT_AMOUNT": 1000000,
    "TAKE_PROFIT": 0.02,
    "STOP_LOSS": -0.01
  }
}
```

### 자동 저장
```python
# engine/base_engine.py
async def save_state_periodically(self):
    """5분마다 자동 저장"""
    while self.running:
        state = {
            'timestamp': time.time(),
            'balance': self.get_current_balance(),
            'positions': self.get_open_positions(),
            'risk_state': self.risk_manager.get_state(),
            'config': self.config.to_dict()
        }
        
        with open('storage/system_state.json', 'w') as f:
            json.dump(state, f, indent=2)
        
        await asyncio.sleep(300)  # 5분
```

### 복구 사용
```python
# engine/base_engine.py
async def recover_from_crash(self):
    """재시작 시 상태 복구"""
    try:
        with open('storage/system_state.json', 'r') as f:
            state = json.load(f)
        
        logger.info("=== 시스템 상태 복구 시작 ===")
        
        # 포지션 복구
        for symbol, pos in state['positions'].items():
            self.restore_position(symbol, pos)
        
        # 리스크 상태 복구
        self.risk_manager.restore_state(state['risk_state'])
        
        logger.info("=== 시스템 상태 복구 완료 ===")
    
    except FileNotFoundError:
        logger.info("복구할 상태 없음, 정상 시작")
```

---

## 관리 및 유지보수

### 디스크 용량 모니터링

```python
import shutil

def check_disk_space():
    """디스크 여유 공간 확인"""
    total, used, free = shutil.disk_usage('/storage')
    
    free_gb = free // (2**30)
    
    if free_gb < 1:  # 1GB 미만
        logger.warning(f"⚠️ 디스크 여유 공간 부족: {free_gb}GB")
        return False
    
    return True
```

### 정리 스크립트

**scripts/cleanup_storage.py**:
```python
import os
from datetime import datetime, timedelta

def cleanup_old_cache():
    """오래된 캐시 삭제"""
    cache_dir = 'storage/cache'
    
    for filename in os.listdir(cache_dir):
        if filename == '.gitkeep':
            continue
        
        filepath = os.path.join(cache_dir, filename)
        
        # 1시간 이상 된 파일 삭제
        if os.path.getmtime(filepath) < time.time() - 3600:
            os.remove(filepath)
            print(f"삭제: {filename}")

def backup_database():
    """데이터베이스 백업"""
    import shutil
    from datetime import datetime
    
    src = 'storage/trades.db'
    
    # 백업 디렉토리
    backup_dir = 'storage/backups'
    os.makedirs(backup_dir, exist_ok=True)
    
    # 날짜별 백업
    date_str = datetime.now().strftime('%Y%m%d')
    dst = f'{backup_dir}/trades_{date_str}.db'
    
    shutil.copy2(src, dst)
    print(f"✅ DB 백업 완료: {dst}")

def remove_old_backups(days=30):
    """오래된 백업 삭제"""
    backup_dir = 'storage/backups'
    cutoff = time.time() - (days * 86400)
    
    for filename in os.listdir(backup_dir):
        filepath = os.path.join(backup_dir, filename)
        
        if os.path.getmtime(filepath) < cutoff:
            os.remove(filepath)
            print(f"삭제: {filename}")

if __name__ == '__main__':
    cleanup_old_cache()
    backup_database()
    remove_old_backups(30)
```

### 자동화 (cron)

**Linux/Mac**:
```bash
# crontab -e
# 매일 새벽 3시 백업 및 정리
0 3 * * * cd /path/to/project && python scripts/cleanup_storage.py
```

**Windows (Task Scheduler)**:
```
작업: cleanup_storage
트리거: 매일 오전 3:00
동작: python scripts/cleanup_storage.py
```

---

## 디렉토리별 용량 가이드

```
storage/
├── historical/    ~500MB-2GB  (6개월 5분봉 기준)
├── cache/         ~1-10MB     (항상 작음)
├── trades.db      ~10-100MB   (1년 거래 기준)
├── backups/       ~300MB-3GB  (30일 백업)
└── system_state   ~1-10KB     (무시 가능)

총 예상 용량: 1-5GB
```

### 용량 절약 팁

1. **Historical 데이터**:
   - 필요한 기간만 유지 (6-12개월)
   - 압축 저장 (gzip)
   ```bash
   gzip storage/historical/*.csv
   ```

2. **데이터베이스**:
   - 정기적 VACUUM
   ```sql
   VACUUM;
   ```
   - 오래된 데이터 아카이브

3. **백업**:
   - 압축 백업
   ```bash
   tar -czf trades_backup.tar.gz storage/trades.db
   ```

---

## .gitignore 설정

```gitignore
# Storage 디렉토리
storage/historical/*.csv
storage/cache/*
!storage/cache/.gitkeep
storage/trades.db
storage/trades.db-journal
storage/system_state.json
storage/backups/

# 단, 구조는 유지
!storage/README.md
!storage/historical/README.md
```

---

## 초기 설정 체크리스트

### 개발 환경
- [ ] `storage/` 디렉토리 생성
- [ ] `storage/historical/` 생성
- [ ] `storage/cache/` 생성 + `.gitkeep`
- [ ] `storage/backups/` 생성
- [ ] 과거 데이터 다운로드 (백테스트용)
- [ ] DB 초기화 (`initialize_database()`)

### 프로덕션 환경
- [ ] 디스크 용량 확인 (최소 5GB)
- [ ] 백업 자동화 설정 (cron/Task Scheduler)
- [ ] 모니터링 설정 (디스크 사용량)
- [ ] 권한 설정 (`chmod 755 storage/`)

---

## 문제 해결

### Q: trades.db가 잠겨있어요
**A**: SQLite 저널 파일 삭제
```bash
rm storage/trades.db-journal
```

### Q: 캐시가 작동하지 않아요
**A**: cache/ 디렉토리 권한 확인
```bash
chmod 755 storage/cache/
```

### Q: 백테스트 데이터가 없어요
**A**: 다운로드 스크립트 실행
```bash
python scripts/download_historical_data.py
```

### Q: 디스크 용량 부족
**A**: 정리 스크립트 실행
```bash
python scripts/cleanup_storage.py
```

---

## README.md (storage/)

```markdown
# Storage Directory

데이터 저장소

## 구조
```
storage/
├── historical/     # 백테스트용 과거 데이터
├── cache/          # 1분 캐시
├── trades.db       # 거래 DB
└── system_state    # 시스템 상태 백업
```

## 초기 설정
```bash
# 디렉토리 생성
mkdir -p storage/historical storage/cache storage/backups

# 데이터베이스 초기화
python -c "from database.models import initialize_database; initialize_database()"

# 과거 데이터 다운로드 (선택)
python scripts/download_historical_data.py
```

## 유지보수
```bash
# 백업
python scripts/cleanup_storage.py

# 용량 확인
du -sh storage/*
```

## 주의사항
- `trades.db`는 중요 데이터이므로 정기 백업 필수
- `historical/` CSV는 용량이 크므로 git에서 제외
- `cache/`는 자동으로 관리되므로 수동 삭제 불필요
```

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 12 (데이터 저장소)  
**검증**: ✅ 완료
</artifact>

<artifact identifier="logs-structure-doc" type="text/markdown" title="13_LOGS_구조설명.md">
# 13_LOGS 디렉토리 구조 설명

> **목적**: 로그 파일 구조 및 관리 방법

---

## 📋 목차
1. [디렉토리 구조](#디렉토리-구조)
2. [로그 레벨 및 분류](#로그-레벨-및-분류)
3. [모드별 로그](#모드별-로그)
4. [로그 포맷](#로그-포맷)
5. [로그 로테이션](#로그-로테이션)
6. [관리 및 분석](#관리-및-분석)

---

## 디렉토리 구조

```
logs/
├── paper/                      # 모의투자 로그
│   ├── 2025-01-15_trades.log      # 거래만
│   ├── 2025-01-15_decisions.log   # AI 판단만
│   ├── 2025-01-15_errors.log      # 에러만
│   ├── 2025-01-15_debug.log       # 전체 (DEBUG 모드)
│   └── README.md
│
├── live/                       # 실거래 로그
│   ├── 2025-01-15_trades.log
│   ├── 2025-01-15_decisions.log
│   ├── 2025-01-15_errors.log
│   ├── 2025-01-15_debug.log
│   └── README.md
│
├── backtest/                   # 백테스트 로그
│   ├── 2025-01-15_backtest.log
│   └── README.md
│
├── reports/                    # 리포트 (monitoring/reporter.py)
│   ├── daily/
│   │   ├── 2025-01-15.txt
│   │   └── 2025-01-16.txt
│   ├── weekly/
│   │   ├── 2025-W03.txt
│   │   └── 2025-W04.txt
│   └── monthly/
│       └── 2025-01.txt
│
└── README.md
```

---

## 로그 레벨 및 분류

### Python Logging 레벨
```python
DEBUG    10  # 상세한 디버깅 정보
INFO     20  # 일반 정보 (거래, 판단)
WARNING  30  # 경고 (리스크 근접)
ERROR    40  # 오류 (복구 가능)
CRITICAL 50  # 치명적 오류 (시스템 중단)
```

### 로그 파일별 레벨

| 파일 | 레벨 | 내용 | 필터 |
|------|------|------|------|
| `trades.log` | INFO | 거래 기록 | `[ENTRY]`, `[EXIT]` |
| `decisions.log` | INFO | AI 판단 | `[AI]` |
| `errors.log` | ERROR+ | 오류 | ERROR, CRITICAL |
| `debug.log` | DEBUG+ | 전체 | 모든 레벨 |

---

## 모드별 로그

### paper/ - 모의투자

**용도**: 전략 테스트 및 검증

**주요 로그**:
```
[2025-01-15 14:32:15] [INFO] [ENTRY] DOGE/USDT @ 0.3821 | Amount: 500,000 KRW | AI: 0.75
[2025-01-15 14:32:16] [INFO] [ORDER] Paper Order Filled: 1006 DOGE
[2025-01-15 16:45:22] [INFO] [EXIT] DOGE/USDT @ 0.3895 | PnL: +1.94% (+9,700 KRW) | Reason: TRAILING_STOP
[2025-01-15 18:00:00] [WARNING] [RISK] Daily Loss: -2.3% / -5.0%
```

**보관 기간**: 30일

---

### live/ - 실거래

**용도**: 실제 자금 거래 기록 (영구 보관)

**주요 로그**:
```
[2025-01-15 14:32:15] [INFO] [ENTRY] DOGE/USDT @ 0.3821 | Amount: 500,000 KRW | AI: 0.75
[2025-01-15 14:32:16] [INFO] [ORDER] Bybit Order ID: 1234567890 | Status: Filled
[2025-01-15 14:32:17] [INFO] [ORDER] Filled: 1006/1006 DOGE | Avg Price: 0.3821
[2025-01-15 16:45:22] [INFO] [EXIT] DOGE/USDT @ 0.3895 | PnL: +1.94% | Reason: TRAILING_STOP
[2025-01-15 16:45:23] [INFO] [ORDER] Bybit Order ID: 1234567891 | Status: Filled
[2025-01-15 18:00:00] [WARNING] [RISK] Daily Loss: -4.2% / -5.0% ⚠️
```

**보관 기간**: 영구 (세금 신고용)

---

### backtest/ - 백테스트

**용도**: 과거 데이터 시뮬레이션

**로그 예시**:
```
[2025-01-15 10:00:00] [INFO] Backtest Started: 2024-01-01 ~ 2024-12-31
[2025-01-15 10:00:05] [INFO] [ENTRY] 2024-01-15 DOGE @ 0.3821
[2025-01-15 10:00:06] [INFO] [EXIT] 2024-01-15 DOGE @ 0.3895 | PnL: +1.94%
...
[2025-01-15 10:05:00] [INFO] Backtest Completed
[2025-01-15 10:05:01] [INFO] Total Trades: 127 | Win Rate: 65.4% | Sharpe: 1.85
```

**보관 기간**: 90일

---

## 로그 포맷

### 기본 포맷
```
[YYYY-MM-DD HH:MM:SS] [LEVEL] [TAG] Message
```

### 태그별 포맷

**[ENTRY] - 진입 로그**:
```
[2025-01-15 14:32:15] [INFO] [ENTRY] {symbol} @ {price} | Amount: {amount} KRW | AI: {confidence}
```

**[EXIT] - 청산 로그**:
```
[2025-01-15 16:45:22] [INFO] [EXIT] {symbol} @ {exit_price} | Entry: {entry_price} | PnL: {pnl}% ({pnl_krw} KRW) | Reason: {reason}
```

**[AI] - AI 판단 로그**:
```
[2025-01-15 14:32:10] [INFO] [AI] {symbol} | Action: {action} | Confidence: {conf} | Reasoning: {reason}
```

**[RISK] - 리스크 이벤트**:
```
[2025-01-15 18:00:00] [WARNING] [RISK] {event_type} | Loss: {loss}% | Resume: {resume_time}
```

**[ORDER] - 주문 상태**:
```
[2025-01-15 14:32:16] [INFO] [ORDER] Bybit Order ID: {order_id} | Status: {status} | Filled: {filled}/{total}
```

**[API] - API 호출**:
```
[2025-01-15 14:32:10] [DEBUG] [API] Bybit fetch_ticker DOGE/USDT | Response Time: 123ms
[2025-01-15 20:15:33] [ERROR] [API] Bybit Connection Timeout - Retrying (3/60)...
```

**[NETWORK] - 네트워크 상태**:
```
[2025-01-15 14:23:00] [WARNING] [NETWORK] Connection Lost - Waiting for Recovery...
[2025-01-15 14:23:30] [INFO] [NETWORK] Connection Restored (30s downtime)
```

---

## 로그 로테이션

### 일별 로테이션 (기본)

**구현** (monitoring/logger.py):
```python
def setup_logger(self) -> logging.Logger:
    today = datetime.now().strftime('%Y-%m-%d')
    
    log_dir = Path(LOG_DIR) / self.mode
    log_dir.mkdir(parents=True, exist_ok=True)
    
    # 거래 로그
    trades_handler = logging.FileHandler(
        log_dir / f'{today}_trades.log',
        encoding='utf-8'
    )
    # ...
```

### TimedRotatingFileHandler (선택)

**자동 로테이션**:
```python
from logging.handlers import TimedRotatingFileHandler

handler = TimedRotatingFileHandler(
    filename='logs/paper/trades.log',
    when='midnight',      # 자정마다
    interval=1,           # 1일
    backupCount=30,       # 30개 보관
    encoding='utf-8'
)
```

### 수동 아카이브

**scripts/archive_logs.py**:
```python
import os
import shutil
from datetime import datetime, timedelta

def archive_old_logs(mode='paper', days=30):
    """30일 이상 된 로그 압축"""
    log_dir = f'logs/{mode}'
    archive_dir = f'logs/archives/{mode}'
    
    os.makedirs(archive_dir, exist_ok=True)
    
    cutoff_date = datetime.now() - timedelta(days=days)
    
    for filename in os.listdir(log_dir):
        if not filename.endswith('.log'):
            continue
        
        filepath = os.path.join(log_dir, filename)
        
        # 파일 수정 시간 확인
        mtime = datetime.fromtimestamp(os.path.getmtime(filepath))
        
        if mtime < cutoff_date:
            # 압축 후 이동
            import gzip
            
            gz_filename = f'{filename}.gz'
            gz_filepath = os.path.join(archive_dir, gz_filename)
            
            with open(filepath, 'rb') as f_in:
                with gzip.open(gz_filepath, 'wb') as f_out:
                    shutil.copyfileobj(f_in, f_out)
            
            # 원본 삭제
            os.remove(filepath)
            
            print(f"✅ 아카이브: {filename} → {gz_filename}")

# 사용
archive_old_logs('paper', days=30)
archive_old_logs('live', days=365)  # Live는 1년 보관
```

---

## 관리 및 분석

### 실시간 모니터링

**Linux/Mac**:
```bash
# 전체 로그 실시간 보기
tail -f logs/paper/2025-01-15_debug.log

# 거래만 보기
tail -f logs/paper/2025-01-15_trades.log

# 에러만 보기
tail -f logs/paper/2025-01-15_errors.log

# 여러 파일 동시 보기
tail -f logs/paper/2025-01-15_*.log
```

**Windows**:
```powershell
# PowerShell
Get-Content logs\paper\2025-01-15_debug.log -Wait -Tail 50
```

### 로그 검색

**오늘 진입 거래 찾기**:
```bash
grep "\[ENTRY\]" logs/paper/$(date +%Y-%m-%d)_trades.log
```

**오늘 손실 거래 찾기**:
```bash
grep "PnL: -" logs/paper/$(date +%Y-%m-%d)_trades.log
```

**특정 심볼 거래**:
```bash
grep "DOGE" logs/paper/$(date +%Y-%m-%d)_trades.log
```

**에러 발생 횟수**:
```bash
grep -c "\[ERROR\]" logs/paper/$(date +%Y-%m-%d)_errors.log
```

### 로그 분석 스크립트

**scripts/analyze_logs.py**:
```python
import re
from collections import defaultdict

def analyze_trades_log(filepath):
    """거래 로그 분석"""
    trades = []
    
    with open(filepath, 'r', encoding='utf-8') as f:
        for line in f:
            if '[EXIT]' in line:
                # PnL 추출
                match = re.search(r'PnL: ([+-]\d+\.\d+)%', line)
                if match:
                    pnl = float(match.group(1))
                    trades.append(pnl)
    
    if not trades:
        print("거래 없음")
        return
    
    # 통계
    total = len(trades)
    winners = [t for t in trades if t > 0]
    losers = [t for t in trades if t <= 0]
    
    win_rate = len(winners) / total * 100
    avg_profit = sum(winners) / len(winners) if winners else 0
    avg_loss = sum(losers) / len(losers) if losers else 0
    
    print(f"총 거래: {total}")
    print(f"승률: {win_rate:.1f}% ({len(winners)}승 {len(losers)}패)")
    print(f"평균 수익: {avg_profit:.2f}%")
    print(f"평균 손실: {avg_loss:.2f}%")
    print(f"최대 수익: {max(trades):.2f}%")
    print(f"최대 손실: {min(trades):.2f}%")

# 사용
analyze_trades_log('logs/paper/2025-01-15_trades.log')
```

**scripts/check_errors.py**:
```python
def check_error_frequency(filepath):
    """에러 빈도 확인"""
    error_types = defaultdict(int)
    
    with open(filepath, 'r', encoding='utf-8') as f:
        for line in f:
            if '[ERROR]' in line or '[CRITICAL]' in line:
                # 에러 타입 추출
                if 'Network' in line:
                    error_types['Network'] += 1
                elif 'API' in line:
                    error_types['API'] += 1
                elif 'Database' in line:
                    error_types['Database'] += 1
                else:
                    error_types['Other'] += 1
    
    if not error_types:
        print("✅ 에러 없음")
        return
    
    print("⚠️ 에러 발생:")
    for error_type, count in sorted(error_types.items(), key=lambda x: -x[1]):
        print(f"  {error_type}: {count}회")

# 사용
check_error_frequency('logs/paper/2025-01-15_errors.log')
```

### 성능 분석

**스크립트 활용**:
```bash
# 오늘 승률
python scripts/analyze_logs.py logs/paper/$(date +%Y-%m-%d)_trades.log

# 에러 체크
python scripts/check_errors.py logs/paper/$(date +%Y-%m-%d)_errors.log

# 주간 통계
for file in logs/paper/*_trades.log; do
    echo "=== $file ==="
    python scripts/analyze_logs.py "$file"
done
```

---

## 디스크 사용량

### 예상 용량

```
일일 로그:
  trades.log:    ~100KB  (하루 10거래 기준)
  decisions.log: ~200KB  (AI 호출 30회)
  errors.log:    ~50KB   (정상 운영 시)
  debug.log:     ~5MB    (DEBUG 모드)

총 일일:         ~5-6MB
월간:            ~150-180MB
년간:            ~2GB
```

### 용량 최적화

1. **DEBUG 모드 비활성화** (프로덕션):
```python
# config.py
LOG_LEVEL = 'INFO'  # DEBUG 대신 INFO
```

2. **로그 압축**:
```bash
# 오래된 로그 압축
gzip logs/paper/*_2024-*.log
```

3. **자동 정리**:
```bash
# crontab
# 매월 1일 90일 이상 로그 삭제
0 0 1 * * find logs/ -name "*.log" -mtime +90 -delete
```

---

## .gitignore 설정

```gitignore
# 로그 파일
logs/*.log
logs/**/*.log
logs/**/*.gz

# 단, 구조는 유지
!logs/README.md
!logs/paper/README.md
!logs/live/README.md
!logs/backtest/README.md
```

---

## README.md (logs/)

```markdown
# Logs Directory

시스템 로그 저장소

## 구조
```
logs/
├── paper/       # 모의투자
├── live/        # 실거래
├── backtest/    # 백테스트
└── reports/     # 리포트
```

## 로그 파일
- `trades.log`: 거래 기록
- `decisions.log`: AI 판단
- `errors.log`: 에러 로그
- `debug.log`: 전체 디버그

## 실시간 모니터링
```bash
# 거래 로그
tail -f logs/paper/$(date +%Y-%m-%d)_trades.log

# 에러 로그
tail -f logs/paper/$(date +%Y-%m-%d)_errors.log
```

## 분석
```bash
# 오늘 통계
python scripts/analyze_logs.py logs/paper/$(date +%Y-%m-%d)_trades.log

# 에러 체크
python scripts/check_errors.py logs/paper/$(date +%Y-%m-%d)_errors.log
```

## 보관 기간
- **Paper**: 30일
- **Live**: 영구 (세금 신고용)
- **Backtest**: 90일
- **Reports**: 1년
```

---

## 자동화 (cron/Task Scheduler)

### Linux/Mac

**crontab -e**:
```bash
# 매일 자정 로그 아카이브
0 0 * * * cd /path/to/project && python scripts/archive_logs.py

# 매월 1일 오래된 로그 삭제
0 0 1 * * find /path/to/project/logs -name "*.log" -mtime +90 -delete

# 매일 오전 9시 일일 리포트
0 9 * * * cd /path/to/project && python -c "from monitoring.reporter import PerformanceReporter; r = PerformanceReporter(); print(r.generate_daily_summary())"
```

### Windows Task Scheduler

```xml
작업 1: 로그 아카이브
  트리거: 매일 자정
  동작: python scripts/archive_logs.py

작업 2: 로그 정리
  트리거: 매월 1일
  동작: powershell "Get-ChildItem logs -Recurse | Where-Object {$_.LastWriteTime -lt (Get-Date).AddDays(-90)} | Remove-Item"
```

---

## 문제 해결

### Q: 로그 파일이 너무 커요
**A**: DEBUG 모드 비활성화 + 로그 압축
```python
# config.py
LOG_LEVEL = 'INFO'
```

### Q: 로그가 기록되지 않아요
**A**: 디렉토리 권한 확인
```bash
chmod 755 logs/
```

### Q: 실시간 로그를 보고 싶어요
**A**: tail 명령 사용
```bash
tail -f logs/paper/$(date +%Y-%m-%d)_trades.log
```

### Q: 특정 날짜 로그를 찾고 싶어요
**A**: 파일명이 날짜 기반이므로 직접 열기
```bash
cat logs/paper/2025-01-10_trades.log
```

---

**문서 버전**: v1.0  
**작성일**: 2025-01-15  
**Phase**: 13 (로그 시스템)  
**검증**: ✅ 완료
</artifact>
