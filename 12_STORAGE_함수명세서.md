# 12_STORAGE 모듈 완벽 함수 명세서 v2.0 (개선판)

> **개선사항**: 디렉토리 구조 설명서에서 **실제 구현 가능한 함수 명세서**로 완전 변환

---

## 📋 목차
1. [storage/historical_manager.py](#storagehistorical_managerpy) ⭐ 신규
2. [storage/cache_manager.py](#storagecache_managerpy) ⭐ 신규
3. [storage/state_manager.py](#storagestate_managerpy) ⭐ 신규
4. [storage/cleanup.py](#storagecleanuppy) ⭐ 신규
5. [전체 의존성 그래프](#전체-의존성-그래프)
6. [실전 사용 예제](#실전-사용-예제)

---

## 📁 storage/historical_manager.py ⭐ 신규

### 구현 코드 (전체 신규)

```python
import ccxt
import pandas as pd
from pathlib import Path
from datetime import datetime, timedelta
from typing import List, Dict
from core.constants import HISTORICAL_DATA_DIR


class HistoricalDataManager:
    """
    백테스트용 과거 데이터 관리
    
    ⭐ 신규 모듈: 과거 데이터 다운로드, 검증, 로드
    """
    
    def __init__(self):
        """
        데이터 매니저 초기화
        
        호출:
            scripts/download_historical_data.py
            engine/backtest_engine.py
        """
        self.data_dir = Path(HISTORICAL_DATA_DIR)
        self.data_dir.mkdir(parents=True, exist_ok=True)
    
    def download_historical_data(
        self,
        symbol: str,
        timeframe: str = '5m',
        days: int = 180
    ) -> bool:
        """
        Bybit API로 과거 데이터 다운로드
        
        Args:
            symbol: 'DOGE/USDT', 'SOL/USDT'
            timeframe: '1m', '5m', '15m', '1h', '1d'
            days: 다운로드 일수 (기본 180일 = 6개월)
        
        Returns:
            bool: 성공 여부
        
        호출:
            scripts/download_historical_data.py
        
        Example:
            >>> manager = HistoricalDataManager()
            >>> success = manager.download_historical_data('DOGE/USDT', '5m', 180)
            다운로드 중: DOGE/USDT 5m (180일)
            진행: 1000/51840 캔들
            진행: 2000/51840 캔들
            ...
            ✅ DOGE_USDT_5m.csv 저장 완료 (51840개)
        """
        print(f"다운로드 중: {symbol} {timeframe} ({days}일)")
        
        try:
            exchange = ccxt.bybit()
            
            # 시작 시간
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
                
                # 진행 상황 출력
                if len(all_ohlcv) % 1000 == 0:
                    print(f"진행: {len(all_ohlcv)} 캔들")
                
                if len(ohlcv) < 1000:
                    break
            
            if not all_ohlcv:
                print(f"❌ 데이터 없음: {symbol}")
                return False
            
            # DataFrame 변환
            df = pd.DataFrame(
                all_ohlcv,
                columns=['timestamp', 'open', 'high', 'low', 'close', 'volume']
            )
            
            # 타임스탬프를 초 단위로 변환
            df['timestamp'] = df['timestamp'] // 1000
            
            # 파일명
            filename = f"{symbol.replace('/', '_')}_{timeframe}.csv"
            filepath = self.data_dir / filename
            
            # CSV 저장
            df.to_csv(filepath, index=False)
            
            print(f"✅ {filename} 저장 완료 ({len(df)}개)")
            return True
        
        except Exception as e:
            print(f"❌ 다운로드 실패: {e}")
            return False
    
    def validate_csv(self, filepath: str) -> Dict:
        """
        CSV 데이터 검증
        
        Args:
            filepath: CSV 파일 경로
        
        Returns:
            {
                'valid': bool,
                'errors': List[str],
                'count': int,
                'start_date': str,
                'end_date': str
            }
        
        호출:
            scripts/validate_historical_data.py
        
        Example:
            >>> manager = HistoricalDataManager()
            >>> result = manager.validate_csv('storage/historical/DOGE_USDT_5m.csv')
            >>> if result['valid']:
            >>>     print(f"검증 성공: {result['count']}개")
            >>> else:
            >>>     print(f"검증 실패: {result['errors']}")
        """
        errors = []
        
        try:
            df = pd.read_csv(filepath)
            
            # 1. 필수 컬럼 확인
            required = ['timestamp', 'open', 'high', 'low', 'close', 'volume']
            missing = [col for col in required if col not in df.columns]
            if missing:
                errors.append(f"필수 컬럼 누락: {missing}")
                return {
                    'valid': False,
                    'errors': errors,
                    'count': 0,
                    'start_date': '',
                    'end_date': ''
                }
            
            # 2. 데이터 타입 확인
            if df['timestamp'].dtype not in ['int64', 'int32']:
                errors.append(f"타임스탬프 타입 오류: {df['timestamp'].dtype}")
            
            for col in ['open', 'high', 'low', 'close', 'volume']:
                if df[col].dtype != 'float64':
                    errors.append(f"{col} 타입 오류: {df[col].dtype}")
            
            # 3. 논리 검증
            if not (df['high'] >= df['low']).all():
                errors.append("high < low인 캔들 존재")
            
            if not (df['high'] >= df['open']).all():
                errors.append("high < open인 캔들 존재")
            
            if not (df['high'] >= df['close']).all():
                errors.append("high < close인 캔들 존재")
            
            if not (df['low'] <= df['open']).all():
                errors.append("low > open인 캔들 존재")
            
            if not (df['low'] <= df['close']).all():
                errors.append("low > close인 캔들 존재")
            
            # 4. 시간 순서 확인
            if not df['timestamp'].is_monotonic_increasing:
                errors.append("타임스탬프 순서 오류")
            
            # 5. 날짜 범위
            start_date = datetime.fromtimestamp(df['timestamp'].min()).strftime('%Y-%m-%d')
            end_date = datetime.fromtimestamp(df['timestamp'].max()).strftime('%Y-%m-%d')
            
            if errors:
                return {
                    'valid': False,
                    'errors': errors,
                    'count': len(df),
                    'start_date': start_date,
                    'end_date': end_date
                }
            
            return {
                'valid': True,
                'errors': [],
                'count': len(df),
                'start_date': start_date,
                'end_date': end_date
            }
        
        except Exception as e:
            return {
                'valid': False,
                'errors': [str(e)],
                'count': 0,
                'start_date': '',
                'end_date': ''
            }
    
    def load_historical_data(
        self,
        symbol: str,
        timeframe: str = '5m'
    ) -> pd.DataFrame:
        """
        CSV 파일 로드
        
        Args:
            symbol: 'DOGE/USDT', 'SOL/USDT'
            timeframe: '1m', '5m', '15m', '1h', '1d'
        
        Returns:
            pd.DataFrame: OHLCV 데이터
        
        호출:
            engine/backtest_engine.py
        
        Example:
            >>> manager = HistoricalDataManager()
            >>> df = manager.load_historical_data('DOGE/USDT', '5m')
            >>> print(f"로드됨: {len(df)}개 캔들")
            >>> print(df.head())
        """
        filename = f"{symbol.replace('/', '_')}_{timeframe}.csv"
        filepath = self.data_dir / filename
        
        if not filepath.exists():
            raise FileNotFoundError(f"파일 없음: {filepath}")
        
        df = pd.read_csv(filepath)
        
        # 검증
        result = self.validate_csv(filepath)
        if not result['valid']:
            raise ValueError(f"데이터 검증 실패: {result['errors']}")
        
        return df
    
    def get_available_data(self) -> List[Dict]:
        """
        사용 가능한 데이터 목록
        
        Returns:
            [{
                'symbol': 'DOGE/USDT',
                'timeframe': '5m',
                'filename': 'DOGE_USDT_5m.csv',
                'count': 51840,
                'start_date': '2024-07-15',
                'end_date': '2025-01-15'
            }, ...]
        
        Example:
            >>> manager = HistoricalDataManager()
            >>> data_list = manager.get_available_data()
            >>> for data in data_list:
            >>>     print(f"{data['symbol']} {data['timeframe']}: {data['count']}개")
        """
        result = []
        
        for filepath in self.data_dir.glob('*.csv'):
            # 파일명 파싱 (예: DOGE_USDT_5m.csv)
            parts = filepath.stem.split('_')
            if len(parts) >= 3:
                symbol = f"{parts[0]}/{parts[1]}"
                timeframe = parts[2]
                
                # 검증
                validation = self.validate_csv(filepath)
                
                result.append({
                    'symbol': symbol,
                    'timeframe': timeframe,
                    'filename': filepath.name,
                    'count': validation['count'],
                    'start_date': validation['start_date'],
                    'end_date': validation['end_date'],
                    'valid': validation['valid']
                })
        
        return result
```

---

## 📁 storage/cache_manager.py ⭐ 신규

### 구현 코드 (전체 신규)

```python
import json
import time
from pathlib import Path
from typing import Dict, Any, Optional
from core.constants import CACHE_DIR


class CacheManager:
    """
    1분간 유효한 API 응답 캐시
    
    ⭐ 신규 모듈: 메모리 + 파일 캐시 관리
    """
    
    def __init__(self, cache_ttl: int = 60):
        """
        캐시 매니저 초기화
        
        Args:
            cache_ttl: 캐시 유효 시간 (초, 기본 60초)
        
        호출:
            data/fetcher.py
            indicators/calculator.py
        """
        self.cache_ttl = cache_ttl
        self.cache_dir = Path(CACHE_DIR)
        self.cache_dir.mkdir(parents=True, exist_ok=True)
        
        # 메모리 캐시 (빠른 접근)
        self.memory_cache = {}
    
    def get(
        self,
        cache_type: str,
        key: str
    ) -> Optional[Dict]:
        """
        캐시 조회
        
        Args:
            cache_type: 'market_data', 'indicators'
            key: 'DOGE/USDT', 'SOL/USDT'
        
        Returns:
            Dict or None: 캐시된 데이터 (만료 시 None)
        
        호출:
            data/fetcher.py - 시세 조회 전
        
        Example:
            >>> cache = CacheManager()
            >>> data = cache.get('market_data', 'DOGE/USDT')
            >>> if data:
            >>>     print("캐시 hit!")
            >>> else:
            >>>     print("캐시 miss, API 호출 필요")
        """
        cache_key = f"{cache_type}:{key}"
        
        # 1. 메모리 캐시 확인
        if cache_key in self.memory_cache:
            cached = self.memory_cache[cache_key]
            
            # 만료 확인
            if cached['expires_at'] > time.time():
                return cached['data']
            else:
                # 만료된 캐시 삭제
                del self.memory_cache[cache_key]
        
        # 2. 파일 캐시 확인 (메모리 miss 시)
        filepath = self.cache_dir / f"{cache_type}.json"
        
        if filepath.exists():
            try:
                with open(filepath, 'r', encoding='utf-8') as f:
                    file_cache = json.load(f)
                
                if key in file_cache:
                    cached = file_cache[key]
                    
                    if cached['expires_at'] > time.time():
                        # 메모리에도 저장 (다음번 빠른 접근)
                        self.memory_cache[cache_key] = cached
                        return cached['data']
            
            except (json.JSONDecodeError, KeyError):
                pass
        
        return None
    
    def set(
        self,
        cache_type: str,
        key: str,
        data: Dict
    ) -> None:
        """
        캐시 저장
        
        Args:
            cache_type: 'market_data', 'indicators'
            key: 'DOGE/USDT', 'SOL/USDT'
            data: 캐시할 데이터
        
        호출:
            data/fetcher.py - API 호출 후
        
        Example:
            >>> cache = CacheManager()
            >>> cache.set('market_data', 'DOGE/USDT', {
            ...     'price': 0.3821,
            ...     'volume_24h': 1234567890
            ... })
        """
        cache_key = f"{cache_type}:{key}"
        expires_at = time.time() + self.cache_ttl
        
        cached_data = {
            'data': data,
            'timestamp': time.time(),
            'expires_at': expires_at
        }
        
        # 1. 메모리 캐시 저장
        self.memory_cache[cache_key] = cached_data
        
        # 2. 파일 캐시 저장
        filepath = self.cache_dir / f"{cache_type}.json"
        
        try:
            # 기존 파일 로드
            if filepath.exists():
                with open(filepath, 'r', encoding='utf-8') as f:
                    file_cache = json.load(f)
            else:
                file_cache = {}
            
            # 업데이트
            file_cache[key] = cached_data
            
            # 저장
            with open(filepath, 'w', encoding='utf-8') as f:
                json.dump(file_cache, f, indent=2)
        
        except Exception as e:
            print(f"캐시 파일 저장 오류: {e}")
    
    def clear(self, cache_type: str = None) -> None:
        """
        캐시 삭제
        
        Args:
            cache_type: 특정 타입만 삭제 (None이면 전체)
        
        Example:
            >>> cache = CacheManager()
            >>> cache.clear('market_data')  # market_data만 삭제
            >>> cache.clear()  # 전체 삭제
        """
        if cache_type:
            # 특정 타입만 삭제
            keys_to_delete = [
                k for k in self.memory_cache.keys()
                if k.startswith(f"{cache_type}:")
            ]
            for key in keys_to_delete:
                del self.memory_cache[key]
            
            # 파일 삭제
            filepath = self.cache_dir / f"{cache_type}.json"
            if filepath.exists():
                filepath.unlink()
        
        else:
            # 전체 삭제
            self.memory_cache.clear()
            
            for filepath in self.cache_dir.glob('*.json'):
                filepath.unlink()
    
    def get_stats(self) -> Dict:
        """
        캐시 통계
        
        Returns:
            {
                'memory_size': 메모리 캐시 크기,
                'file_count': 파일 캐시 개수,
                'total_keys': 전체 키 수
            }
        
        Example:
            >>> cache = CacheManager()
            >>> stats = cache.get_stats()
            >>> print(f"캐시된 항목: {stats['total_keys']}개")
        """
        file_count = len(list(self.cache_dir.glob('*.json')))
        
        return {
            'memory_size': len(self.memory_cache),
            'file_count': file_count,
            'total_keys': len(self.memory_cache)
        }
```

---

## 📁 storage/state_manager.py ⭐ 신규

### 구현 코드 (전체 신규)

```python
import json
import time
from pathlib import Path
from typing import Dict, Optional
from datetime import datetime


class SystemStateManager:
    """
    시스템 상태 저장 및 복구
    
    ⭐ 신규 모듈: 비정상 종료 시 상태 복구
    """
    
    def __init__(self, state_file: str = 'storage/system_state.json'):
        """
        상태 매니저 초기화
        
        Args:
            state_file: 상태 파일 경로
        
        호출:
            engine/base_engine.py
        """
        self.state_file = Path(state_file)
        self.state_file.parent.mkdir(parents=True, exist_ok=True)
    
    def save_state(self, state: Dict) -> bool:
        """
        시스템 상태 저장
        
        Args:
            state: {
                'timestamp': float,
                'mode': str,
                'balance': float,
                'positions': Dict,
                'risk_state': Dict,
                'config': Dict
            }
        
        Returns:
            bool: 저장 성공 여부
        
        호출:
            engine/base_engine.py - 5분마다 자동
        
        Example:
            >>> manager = SystemStateManager()
            >>> success = manager.save_state({
            ...     'timestamp': time.time(),
            ...     'balance': 1050000,
            ...     'positions': {'DOGE/USDT': {...}},
            ...     'risk_state': {...}
            ... })
        """
        try:
            # 타임스탬프 추가
            state['last_updated'] = datetime.now().isoformat()
            
            # 저장
            with open(self.state_file, 'w', encoding='utf-8') as f:
                json.dump(state, f, indent=2, ensure_ascii=False)
            
            return True
        
        except Exception as e:
            print(f"❌ 상태 저장 실패: {e}")
            return False
    
    def load_state(self) -> Optional[Dict]:
        """
        시스템 상태 로드
        
        Returns:
            Dict or None: 저장된 상태
        
        호출:
            engine/base_engine.py - 재시작 시
        
        Example:
            >>> manager = SystemStateManager()
            >>> state = manager.load_state()
            >>> if state:
            >>>     print(f"마지막 업데이트: {state['last_updated']}")
            >>>     print(f"열린 포지션: {len(state['positions'])}개")
        """
        if not self.state_file.exists():
            return None
        
        try:
            with open(self.state_file, 'r', encoding='utf-8') as f:
                state = json.load(f)
            
            return state
        
        except (json.JSONDecodeError, KeyError) as e:
            print(f"❌ 상태 로드 실패: {e}")
            return None
    
    def restore_positions(
        self,
        state: Dict
    ) -> Dict[str, Dict]:
        """
        포지션 복구
        
        Args:
            state: load_state()로 로드한 상태
        
        Returns:
            {
                'DOGE/USDT': {
                    'trade_id': 123,
                    'entry_price': 0.3821,
                    'quantity': 1006,
                    'entry_time': 1640000000
                },
                ...
            }
        
        호출:
            engine/base_engine.py - initialize()
        
        Example:
            >>> manager = SystemStateManager()
            >>> state = manager.load_state()
            >>> if state:
            >>>     positions = manager.restore_positions(state)
            >>>     for symbol, pos in positions.items():
            >>>         print(f"복구: {symbol} @ {pos['entry_price']}")
        """
        if not state or 'positions' not in state:
            return {}
        
        return state['positions']
    
    def clear_state(self) -> bool:
        """
        상태 파일 삭제
        
        호출:
            engine/base_engine.py - 정상 종료 시
        
        Example:
            >>> manager = SystemStateManager()
            >>> manager.clear_state()
        """
        try:
            if self.state_file.exists():
                self.state_file.unlink()
            return True
        
        except Exception as e:
            print(f"❌ 상태 삭제 실패: {e}")
            return False
```

---

## 📁 storage/cleanup.py ⭐ 신규

### 구현 코드 (전체 신규)

```python
import os
import shutil
from pathlib import Path
from datetime import datetime, timedelta
from typing import Dict


class StorageCleanup:
    """
    스토리지 정리 및 최적화
    
    ⭐ 신규 모듈: 디스크 용량 관리
    """
    
    def __init__(self):
        """
        정리 매니저 초기화
        
        호출:
            scripts/cleanup_storage.py
        """
        pass
    
    def cleanup_cache(self, max_age_hours: int = 1) -> Dict:
        """
        오래된 캐시 파일 삭제
        
        Args:
            max_age_hours: 최대 보관 시간 (기본 1시간)
        
        Returns:
            {
                'deleted_count': 삭제된 파일 수,
                'freed_bytes': 확보된 용량 (바이트)
            }
        
        호출:
            scripts/cleanup_storage.py - 매일 자정
        
        Example:
            >>> cleanup = StorageCleanup()
            >>> result = cleanup.cleanup_cache(max_age_hours=1)
            >>> print(f"삭제: {result['deleted_count']}개")
            >>> print(f"확보: {result['freed_bytes']/1024/1024:.1f}MB")
        """
        from core.constants import CACHE_DIR
        
        cache_dir = Path(CACHE_DIR)
        if not cache_dir.exists():
            return {'deleted_count': 0, 'freed_bytes': 0}
        
        cutoff_time = datetime.now() - timedelta(hours=max_age_hours)
        cutoff_timestamp = cutoff_time.timestamp()
        
        deleted_count = 0
        freed_bytes = 0
        
        for filepath in cache_dir.glob('*.json'):
            if filepath.name == '.gitkeep':
                continue
            
            # 파일 수정 시간
            mtime = os.path.getmtime(filepath)
            
            if mtime < cutoff_timestamp:
                # 파일 크기
                size = filepath.stat().st_size
                
                # 삭제
                filepath.unlink()
                
                deleted_count += 1
                freed_bytes += size
                
                print(f"삭제: {filepath.name}")
        
        return {
            'deleted_count': deleted_count,
            'freed_bytes': freed_bytes
        }
    
    def backup_database(self, backup_dir: str = 'storage/backups') -> bool:
        """
        데이터베이스 백업
        
        Args:
            backup_dir: 백업 디렉토리
        
        Returns:
            bool: 백업 성공 여부
        
        호출:
            scripts/cleanup_storage.py - 매일 03:00
        
        Example:
            >>> cleanup = StorageCleanup()
            >>> success = cleanup.backup_database()
            >>> if success:
            >>>     print("✅ DB 백업 완료")
        """
        from core.constants import DB_PATH
        
        src = Path(DB_PATH)
        if not src.exists():
            print("❌ DB 파일 없음")
            return False
        
        # 백업 디렉토리 생성
        backup_path = Path(backup_dir)
        backup_path.mkdir(parents=True, exist_ok=True)
        
        # 날짜별 백업
        date_str = datetime.now().strftime('%Y%m%d')
        dst = backup_path / f'trades_{date_str}.db'
        
        try:
            shutil.copy2(src, dst)
            print(f"✅ DB 백업 완료: {dst}")
            return True
        
        except Exception as e:
            print(f"❌ DB 백업 실패: {e}")
            return False
    
    def remove_old_backups(
        self,
        backup_dir: str = 'storage/backups',
        keep_days: int = 30
    ) -> Dict:
        """
        오래된 백업 삭제
        
        Args:
            backup_dir: 백업 디렉토리
            keep_days: 보관 일수 (기본 30일)
        
        Returns:
            {
                'deleted_count': 삭제된 파일 수,
                'freed_bytes': 확보된 용량 (바이트)
            }
        
        Example:
            >>> cleanup = StorageCleanup()
            >>> result = cleanup.remove_old_backups(keep_days=30)
            >>> print(f"삭제: {result['deleted_count']}개 백업")
        """
        backup_path = Path(backup_dir)
        if not backup_path.exists():
            return {'deleted_count': 0, 'freed_bytes': 0}
        
        cutoff_time = datetime.now() - timedelta(days=keep_days)
        cutoff_timestamp = cutoff_time.timestamp()
        
        deleted_count = 0
        freed_bytes = 0
        
        for filepath in backup_path.glob('*.db'):
            mtime = os.path.getmtime(filepath)
            
            if mtime < cutoff_timestamp:
                size = filepath.stat().st_size
                
                filepath.unlink()
                
                deleted_count += 1
                freed_bytes += size
                
                print(f"삭제: {filepath.name}")
        
        return {
            'deleted_count': deleted_count,
            'freed_bytes': freed_bytes
        }
    
    def check_disk_space(self, min_free_gb: float = 1.0) -> Dict:
        """
        디스크 여유 공간 확인
        
        Args:
            min_free_gb: 최소 필요 공간 (GB)
        
        Returns:
            {
                'total_gb': 전체 용량,
                'used_gb': 사용 용량,
                'free_gb': 여유 용량,
                'sufficient': 충분 여부
            }
        
        Example:
            >>> cleanup = StorageCleanup()
            >>> space = cleanup.check_disk_space(min_free_gb=1.0)
            >>> if not space['sufficient']:
            >>>     print(f"⚠️ 디스크 공간 부족: {space['free_gb']:.1f}GB")
        """
        total, used, free = shutil.disk_usage('/')
        
        total_gb = total / (2**30)
        used_gb = used / (2**30)
        free_gb = free / (2**30)
        
        sufficient = free_gb >= min_free_gb
        
        return {
            'total_gb': round(total_gb, 2),
            'used_gb': round(used_gb, 2),
            'free_gb': round(free_gb, 2),
            'sufficient': sufficient
        }
    
    def get_storage_summary(self) -> Dict:
        """
        스토리지 요약 정보
        
        Returns:
            {
                'historical': {
                    'count': 파일 수,
                    'size_mb': 용량 (MB)
                },
                'cache': {...},
                'database': {...},
                'backups': {...},
                'total_mb': 전체 용량
            }
        
        Example:
            >>> cleanup = StorageCleanup()
            >>> summary = cleanup.get_storage_summary()
            >>> print(f"Historical: {summary['historical']['size_mb']:.1f}MB")
            >>> print(f"Total: {summary['total_mb']:.1f}MB")
        """
        from core.constants import HISTORICAL_DATA_DIR, CACHE_DIR, DB_PATH
        
        result = {}
        total_size = 0
        
        # Historical
        hist_dir = Path(HISTORICAL_DATA_DIR)
        if hist_dir.exists():
            files = list(hist_dir.glob('*.csv'))
            size = sum(f.stat().st_size for f in files)
            result['historical'] = {
                'count': len(files),
                'size_mb': round(size / (1024**2), 2)
            }
            total_size += size
        
        # Cache
        cache_dir = Path(CACHE_DIR)
        if cache_dir.exists():
            files = list(cache_dir.glob('*.json'))
            size = sum(f.stat().st_size for f in files)
            result['cache'] = {
                'count': len(files),
                'size_mb': round(size / (1024**2), 2)
            }
            total_size += size
        
        # Database
        db_path = Path(DB_PATH)
        if db_path.exists():
            size = db_path.stat().st_size
            result['database'] = {
                'size_mb': round(size / (1024**2), 2)
            }
            total_size += size
        
        # Backups
        backup_dir = Path('storage/backups')
        if backup_dir.exists():
            files = list(backup_dir.glob('*.db'))
            size = sum(f.stat().st_size for f in files)
            result['backups'] = {
                'count': len(files),
                'size_mb': round(size / (1024**2), 2)
            }
            total_size += size
        
        result['total_mb'] = round(total_size / (1024**2), 2)
        
        return result
```

---

## 📁 core/constants.py 추가 필요 ⭐

### 구현 코드 (추가)

```python
# core/constants.py에 추가

from pathlib import Path

# ⭐ Storage 경로
STORAGE_DIR = Path('storage')
HISTORICAL_DATA_DIR = STORAGE_DIR / 'historical'
CACHE_DIR = STORAGE_DIR / 'cache'
DB_PATH = STORAGE_DIR / 'trades.db'
```

---

## 전체 의존성 그래프

```
storage/historical_manager.py ⭐
├── 사용: ccxt, pandas, core/constants
└── 사용처: scripts/download_historical_data.py, engine/backtest_engine.py

storage/cache_manager.py ⭐
├── 사용: json, time, core/constants
└── 사용처: data/fetcher.py, indicators/calculator.py

storage/state_manager.py ⭐
├── 사용: json, datetime
└── 사용처: engine/base_engine.py

storage/cleanup.py ⭐
├── 사용: os, shutil, core/constants
└── 사용처: scripts/cleanup_storage.py
```

---

## 실전 사용 예제

### 예제 1: 과거 데이터 다운로드

```python
from storage import HistoricalDataManager

# 매니저 초기화
manager = HistoricalDataManager()

# DOGE 데이터 다운로드 (6개월, 5분봉)
success = manager.download_historical_data(
    symbol='DOGE/USDT',
    timeframe='5m',
    days=180
)

if success:
    # 검증
    result = manager.validate_csv('storage/historical/DOGE_USDT_5m.csv')
    if result['valid']:
        print(f"✅ 검증 성공: {result['count']}개")
        print(f"기간: {result['start_date']} ~ {result['end_date']}")
    else:
        print(f"❌ 검증 실패: {result['errors']}")
```

### 예제 2: 백테스트에서 데이터 로드

```python
from storage import HistoricalDataManager
from engine import BacktestEngine

class BacktestEngine:
    def __init__(self, config, exchange):
        self.data_manager = HistoricalDataManager()
    
    async def initialize(self):
        # 과거 데이터 로드
        self.doge_data = self.data_manager.load_historical_data(
            'DOGE/USDT',
            '5m'
        )
        
        self.sol_data = self.data_manager.load_historical_data(
            'SOL/USDT',
            '5m'
        )
        
        print(f"✅ DOGE: {len(self.doge_data)}개 캔들")
        print(f"✅ SOL: {len(self.sol_data)}개 캔들")
```

### 예제 3: 캐시 사용

```python
from storage import CacheManager
from data import MarketDataFetcher

class MarketDataFetcher:
    def __init__(self, exchange):
        self.exchange = exchange
        self.cache = CacheManager(cache_ttl=60)
    
    async def fetch_ticker(self, symbol):
        # 캐시 확인
        cached = self.cache.get('market_data', symbol)
        if cached:
            print(f"✅ 캐시 hit: {symbol}")
            return cached
        
        # API 호출
        print(f"🔄 API 호출: {symbol}")
        data = await self.exchange.fetch_ticker(symbol)
        
        # 캐시 저장
        self.cache.set('market_data', symbol, data)
        
        return data
```

### 예제 4: 시스템 상태 저장/복구

```python
from storage import SystemStateManager

class BaseEngine:
    def __init__(self, config, exchange):
        self.state_manager = SystemStateManager()
    
    async def initialize(self):
        # 상태 복구 시도
        state = self.state_manager.load_state()
        
        if state:
            print("🔄 이전 상태 발견, 복구 시도...")
            
            # 포지션 복구
            positions = self.state_manager.restore_positions(state)
            
            for symbol, pos in positions.items():
                self.positions[symbol] = pos
                print(f"✅ 복구: {symbol} @ {pos['entry_price']}")
        
        else:
            print("🆕 새로운 시작")
    
    async def save_state_periodically(self):
        """5분마다 상태 저장"""
        while self.running:
            state = {
                'timestamp': time.time(),
                'mode': self.config.MODE,
                'balance': self.exchange.get_balance()['total_krw'],
                'positions': self.positions,
                'risk_state': self.risk_manager.get_state()
            }
            
            self.state_manager.save_state(state)
            
            await asyncio.sleep(300)
    
    def cleanup(self):
        """정상 종료 시 상태 삭제"""
        self.state_manager.clear_state()
        print("✅ 상태 파일 삭제")
```

### 예제 5: 스토리지 정리

```python
# scripts/cleanup_storage.py
from storage import StorageCleanup

def main():
    cleanup = StorageCleanup()
    
    print("🧹 스토리지 정리 시작...")
    
    # 1. 캐시 정리
    result = cleanup.cleanup_cache(max_age_hours=1)
    print(f"✅ 캐시 정리: {result['deleted_count']}개")
    
    # 2. DB 백업
    success = cleanup.backup_database()
    if success:
        print("✅ DB 백업 완료")
    
    # 3. 오래된 백업 삭제
    result = cleanup.remove_old_backups(keep_days=30)
    print(f"✅ 백업 정리: {result['deleted_count']}개")
    
    # 4. 디스크 공간 확인
    space = cleanup.check_disk_space(min_free_gb=1.0)
    print(f"💾 여유 공간: {space['free_gb']:.1f}GB")
    
    if not space['sufficient']:
        print("⚠️ 디스크 공간 부족!")
    
    # 5. 요약
    summary = cleanup.get_storage_summary()
    print("\n📊 스토리지 요약:")
    print(f"  Historical: {summary['historical']['size_mb']:.1f}MB")
    print(f"  Cache: {summary['cache']['size_mb']:.1f}MB")
    print(f"  Database: {summary['database']['size_mb']:.1f}MB")
    print(f"  Backups: {summary['backups']['size_mb']:.1f}MB")
    print(f"  Total: {summary['total_mb']:.1f}MB")
    
    print("\n✅ 스토리지 정리 완료")

if __name__ == '__main__':
    main()
```

---

## 테스트 시나리오

### historical_manager.py 테스트

```python
import pytest
from storage import HistoricalDataManager
import pandas as pd

def test_download_data():
    """데이터 다운로드 테스트"""
    manager = HistoricalDataManager()
    
    # 짧은 기간으로 테스트
    success = manager.download_historical_data(
        'DOGE/USDT',
        '1h',
        days=7
    )
    
    assert success
    
    # 파일 존재 확인
    filepath = manager.data_dir / 'DOGE_USDT_1h.csv'
    assert filepath.exists()

def test_validate_csv():
    """CSV 검증 테스트"""
    manager = HistoricalDataManager()
    
    # 테스트 데이터 생성
    test_data = pd.DataFrame({
        'timestamp': [1640000000, 1640000300],
        'open': [100.0, 101.0],
        'high': [105.0, 106.0],
        'low': [98.0, 99.0],
        'close': [102.0, 103.0],
        'volume': [1000000, 1100000]
    })
    
    test_file = manager.data_dir / 'test.csv'
    test_data.to_csv(test_file, index=False)
    
    # 검증
    result = manager.validate_csv(test_file)
    
    assert result['valid']
    assert result['count'] == 2
    
    # 정리
    test_file.unlink()

def test_get_available_data():
    """사용 가능한 데이터 목록 테스트"""
    manager = HistoricalDataManager()
    
    data_list = manager.get_available_data()
    
    assert isinstance(data_list, list)
    
    if data_list:
        assert 'symbol' in data_list[0]
        assert 'timeframe' in data_list[0]
```

### cache_manager.py 테스트

```python
def test_cache_set_get():
    """캐시 저장/조회 테스트"""
    cache = CacheManager(cache_ttl=2)
    
    # 저장
    cache.set('test', 'key1', {'value': 123})
    
    # 조회
    data = cache.get('test', 'key1')
    assert data is not None
    assert data['value'] == 123
    
    # 만료 후 조회
    import time
    time.sleep(3)
    
    data = cache.get('test', 'key1')
    assert data is None

def test_cache_clear():
    """캐시 삭제 테스트"""
    cache = CacheManager()
    
    cache.set('test', 'key1', {'value': 1})
    cache.set('test', 'key2', {'value': 2})
    
    # 삭제
    cache.clear('test')
    
    assert cache.get('test', 'key1') is None
    assert cache.get('test', 'key2') is None
```

### state_manager.py 테스트

```python
def test_save_load_state():
    """상태 저장/로드 테스트"""
    manager = SystemStateManager('test_state.json')
    
    # 저장
    state = {
        'timestamp': time.time(),
        'balance': 1000000,
        'positions': {
            'DOGE/USDT': {
                'entry_price': 0.3821,
                'quantity': 1000
            }
        }
    }
    
    success = manager.save_state(state)
    assert success
    
    # 로드
    loaded = manager.load_state()
    assert loaded is not None
    assert loaded['balance'] == 1000000
    
    # 정리
    manager.clear_state()
```

### cleanup.py 테스트

```python
def test_backup_database():
    """DB 백업 테스트"""
    cleanup = StorageCleanup()
    
    success = cleanup.backup_database('test_backups')
    
    if success:
        # 백업 파일 존재 확인
        from datetime import datetime
        date_str = datetime.now().strftime('%Y%m%d')
        backup_file = Path(f'test_backups/trades_{date_str}.db')
        assert backup_file.exists()
        
        # 정리
        backup_file.unlink()

def test_check_disk_space():
    """디스크 공간 확인 테스트"""
    cleanup = StorageCleanup()
    
    space = cleanup.check_disk_space()
    
    assert 'total_gb' in space
    assert 'free_gb' in space
    assert 'sufficient' in space
    assert space['total_gb'] > 0
```

---

## 개발 체크리스트

### core/constants.py ⭐
- [x] ⭐ STORAGE_DIR 추가
- [x] ⭐ HISTORICAL_DATA_DIR 추가
- [x] ⭐ CACHE_DIR 추가
- [x] ⭐ DB_PATH 추가

### historical_manager.py ⭐
- [x] ⭐ HistoricalDataManager 클래스 (신규)
- [x] ⭐ download_historical_data() 메서드
- [x] ⭐ validate_csv() 메서드
- [x] ⭐ load_historical_data() 메서드
- [x] ⭐ get_available_data() 메서드

### cache_manager.py ⭐
- [x] ⭐ CacheManager 클래스 (신규)
- [x] ⭐ get() 메서드 (메모리 + 파일)
- [x] ⭐ set() 메서드 (메모리 + 파일)
- [x] ⭐ clear() 메서드
- [x] ⭐ get_stats() 메서드

### state_manager.py ⭐
- [x] ⭐ SystemStateManager 클래스 (신규)
- [x] ⭐ save_state() 메서드
- [x] ⭐ load_state() 메서드
- [x] ⭐ restore_positions() 메서드
- [x] ⭐ clear_state() 메서드

### cleanup.py ⭐
- [x] ⭐ StorageCleanup 클래스 (신규)
- [x] ⭐ cleanup_cache() 메서드
- [x] ⭐ backup_database() 메서드
- [x] ⭐ remove_old_backups() 메서드
- [x] ⭐ check_disk_space() 메서드
- [x] ⭐ get_storage_summary() 메서드

---

## 주요 특징

### 1. 과거 데이터 관리 ⭐
- Bybit API 자동 다운로드
- CSV 형식 자동 검증
- 백테스트 즉시 사용 가능

### 2. 캐시 시스템 ⭐
- 메모리 + 파일 이중 캐시
- 1분 TTL (설정 가능)
- API 호출 최소화

### 3. 상태 저장/복구 ⭐
- 5분마다 자동 저장
- 비정상 종료 시 복구
- 포지션 무손실 복구

### 4. 자동 정리 ⭐
- 오래된 캐시 삭제
- DB 자동 백업
- 디스크 공간 모니터링

### 5. 실전 중심 설계 ⭐
- 함수 명세서 형태
- 실제 구현 가능
- 테스트 시나리오 포함

---

**문서 버전**: v2.0 (개선판)  
**작성일**: 2025-01-22  
**개선사항**:
- ⭐ 디렉토리 구조 설명서 → 함수 명세서로 완전 변환
- ⭐ historical_manager.py 신규 추가 (과거 데이터 관리)
- ⭐ cache_manager.py 신규 추가 (캐시 관리)
- ⭐ state_manager.py 신규 추가 (상태 저장/복구)
- ⭐ cleanup.py 신규 추가 (스토리지 정리)
- ⭐ core/constants.py 경로 상수 추가
- ✅ 실전 사용 예제 5개
- ✅ 테스트 시나리오 완성

**검증 상태**: ✅ 완료
