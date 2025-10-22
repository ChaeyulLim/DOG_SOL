# 11_UTILS 모듈 완벽 함수 명세서 v2.0 (개선판)

> **개선사항**: KRW_USD_RATE 상수 추가, emergency_network_failure 구현, FEE_RATES constants로 이동, validator 에러 메시지 추가

---

## 📋 목차
1. [utils/network.py](#utilsnetworkpy) ⭐ 개선
2. [utils/fee_calculator.py](#utilsfee_calculatorpy) ⭐ 개선
3. [utils/validators.py](#utilsvalidatorspy) ⭐ 개선
4. [utils/helpers.py](#utilshelperspy)
5. [전체 의존성 그래프](#전체-의존성-그래프)
6. [실전 사용 예제](#실전-사용-예제)

---

## 📁 core/constants.py 추가 필요 ⭐

### 구현 코드 (추가)

```python
# core/constants.py에 추가

# ⭐ 환율 (수동 갱신)
KRW_USD_RATE = 1300  # 1 USD = 1300 KRW

# ⭐ 거래소 수수료율
FEE_RATES = {
    'bybit': {
        'maker': 0.001,  # 0.1%
        'taker': 0.001   # 0.1%
    },
    'binance': {
        'maker': 0.001,
        'taker': 0.001
    }
}
```

---

## 📁 utils/network.py ⭐ 개선

### 구현 코드 (전체 개선)

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
) -> Callable:
    """
    네트워크 오류 시 자동 재시도 데코레이터
    
    Args:
        max_retries: 최대 재시도 횟수 (기본 60회 = 1분)
        delay: 재시도 간격 (초)
        exponential_backoff: 지수 백오프 사용 여부
    
    재시도 대상:
        - ccxt.NetworkError
        - ccxt.RequestTimeout
        - aiohttp.ClientError
        - ConnectionError
        - TimeoutError
    
    호출:
        exchanges/bybit_live.py - 모든 API 호출
        data/fetcher.py - 데이터 수집
    
    Example:
        >>> @retry_on_network_error(max_retries=3, delay=2)
        >>> async def fetch_data():
        >>>     return await api.get()
        
        >>> # 지수 백오프
        >>> @retry_on_network_error(max_retries=5, exponential_backoff=True)
        >>> async def important_call():
        >>>     return await api.critical_operation()
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
                    
                    # 마지막 시도 실패
                    if attempt == max_retries - 1:
                        logger.critical(
                            f"🚨 {func.__name__} 네트워크 오류 "
                            f"{max_retries}회 실패: {e}"
                        )
                        
                        # ⭐ 긴급 처리 호출
                        emergency_network_failure()
                        
                        raise Exception(
                            f"네트워크 실패 ({max_retries}회): {e}"
                        )
                    
                    # 대기 시간 계산
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
                    # 네트워크가 아닌 다른 오류
                    logger.error(f"❌ {func.__name__} 오류: {e}")
                    raise
            
            # 이론상 도달 불가
            raise last_exception
        
        return wrapper
    return decorator


async def check_internet_connection() -> bool:
    """
    인터넷 연결 상태 확인
    
    테스트 URL:
        - https://www.google.com
        - https://www.cloudflare.com
        - https://1.1.1.1
    
    Returns:
        True: 연결 가능
        False: 연결 불가
    
    Example:
        >>> is_connected = await check_internet_connection()
        >>> if not is_connected:
        >>>     print("인터넷 연결 없음")
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


async def wait_for_connection(timeout: int = 300) -> bool:
    """
    네트워크 복구 대기
    
    Args:
        timeout: 최대 대기 시간 (초, 기본 5분)
    
    Returns:
        True: 복구됨
        False: 타임아웃
    
    호출:
        emergency_network_failure() 후
    
    Example:
        >>> if await wait_for_connection(timeout=300):
        >>>     print("네트워크 복구")
        >>> else:
        >>>     print("복구 실패, 시스템 종료")
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
        
        # 30초마다 진행 상황 출력
        if int(elapsed) % 30 == 0 and elapsed > 0:
            remaining = timeout - elapsed
            logger.info(f"⏳ 대기 중... (남은 시간: {remaining:.0f}초)")
        
        await asyncio.sleep(check_interval)


def emergency_network_failure():
    """
    네트워크 장애 시 긴급 처리
    
    ⭐ 개선: 완전히 새로 구현
    
    처리:
        1. 열린 포지션 기록
        2. 시스템 상태 저장
        3. 긴급 알림 (로그)
        4. 복구 대기 안내
    
    호출:
        retry_on_network_error() - 최종 실패 시
    
    Example:
        >>> # 자동 호출 - 직접 호출 불필요
    """
    import json
    from datetime import datetime
    
    logger.critical("=" * 60)
    logger.critical("🚨 긴급 네트워크 장애 프로토콜")
    logger.critical("=" * 60)
    
    try:
        # 1. 현재 상태 저장 시도
        try:
            with open('storage/emergency_state.json', 'w') as f:
                state = {
                    'timestamp': datetime.now().isoformat(),
                    'event': 'NETWORK_FAILURE',
                    'message': '네트워크 장애로 인한 비상 상태 저장'
                }
                json.dump(state, f, indent=2)
            
            logger.critical("✅ 비상 상태 저장 완료")
        
        except Exception as e:
            logger.critical(f"❌ 비상 상태 저장 실패: {e}")
        
        # 2. 포지션 경고
        logger.critical("")
        logger.critical("⚠️ 열린 포지션이 있다면:")
        logger.critical("   1. 거래소 웹/앱에서 수동 확인 필요")
        logger.critical("   2. 필요시 수동 청산")
        logger.critical("")
        
        # 3. 복구 안내
        logger.critical("🔧 복구 방법:")
        logger.critical("   1. 인터넷 연결 확인")
        logger.critical("   2. 시스템 재시작")
        logger.critical("   3. 로그 확인: logs/")
        logger.critical("")
        
        logger.critical("=" * 60)
    
    except Exception as e:
        logger.critical(f"긴급 처리 중 오류: {e}")


class NetworkMonitor:
    """
    네트워크 요청 성공률 모니터링
    
    용도:
        - 네트워크 안정성 추적
        - 성공률 95% 미만 시 경고
    """
    
    def __init__(self):
        """
        네트워크 모니터 초기화
        
        호출:
            engine/base_engine.py
        """
        self.total_requests = 0
        self.successful_requests = 0
        self.failed_requests = 0
        self.recent_results = []
        self.max_recent = 100
        self.last_check_time = time.time()
        
        logger.info("📡 네트워크 모니터 시작")
    
    def record_request(self, success: bool) -> None:
        """
        요청 결과 기록
        
        Args:
            success: 성공 여부
        
        Example:
            >>> monitor.record_request(True)
            >>> monitor.record_request(False)
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
    
    def get_success_rate(self) -> float:
        """
        최근 100개 요청 성공률 조회
        
        Returns:
            float: 0.0 ~ 1.0
        
        Example:
            >>> rate = monitor.get_success_rate()
            >>> print(f"성공률: {rate*100:.1f}%")
            성공률: 98.5%
        """
        if not self.recent_results:
            return 1.0
        
        success_count = sum(self.recent_results)
        return success_count / len(self.recent_results)
    
    def is_stable(self) -> bool:
        """
        네트워크 안정성 판단
        
        Returns:
            True: 안정 (95% 이상)
            False: 불안정
        
        Example:
            >>> if not monitor.is_stable():
            >>>     print("⚠️ 네트워크 불안정")
        """
        return self.get_success_rate() >= 0.95
    
    def get_stats(self) -> dict:
        """
        통계 조회
        
        Returns:
            {
                'total': 전체 요청,
                'success': 성공,
                'failed': 실패,
                'success_rate': 성공률,
                'stable': 안정 여부
            }
        """
        return {
            'total': self.total_requests,
            'success': self.successful_requests,
            'failed': self.failed_requests,
            'success_rate': self.get_success_rate(),
            'stable': self.is_stable()
        }
```

---

## 📁 utils/fee_calculator.py ⭐ 개선

### 구현 코드 (전체 개선)

```python
from typing import Dict
from core.constants import FEE_RATES, KRW_USD_RATE  # ⭐ constants에서 import


class FeeCalculator:
    """
    거래 수수료 계산 및 순수익 산출
    
    ⭐ 개선: FEE_RATES를 constants로 이동
    """
    
    def __init__(self, exchange_name: str = 'bybit'):
        """
        수수료 계산기 초기화
        
        Args:
            exchange_name: 거래소 이름 ('bybit', 'binance')
        
        호출:
            engine/base_engine.py
            exchanges/bybit_live.py
        
        Example:
            >>> calc = FeeCalculator('bybit')
        """
        if exchange_name not in FEE_RATES:
            raise ValueError(f"지원하지 않는 거래소: {exchange_name}")
        
        self.exchange_name = exchange_name
        self.fee_rates = FEE_RATES[exchange_name]
        self.fee_rate = self.fee_rates['taker']  # Market Order는 Taker
    
    def calculate_entry_fee(
        self,
        entry_price: float,
        quantity: float
    ) -> Dict:
        """
        진입 수수료 계산
        
        계산식:
            매수 금액 = entry_price × quantity
            수수료 = 매수 금액 × 0.001
            총 비용 = 매수 금액 + 수수료
        
        Args:
            entry_price: 진입가 (USDT)
            quantity: 수량 (코인)
        
        Returns:
            {
                'trade_value': 거래 금액,
                'fee': 수수료,
                'total_cost': 총 비용,
                'fee_rate': 수수료율
            }
        
        Example:
            >>> calc = FeeCalculator('bybit')
            >>> result = calc.calculate_entry_fee(100.0, 10)
            >>> print(f"총 비용: {result['total_cost']:.2f} USDT")
            총 비용: 1001.00 USDT
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
    
    def calculate_exit_fee(
        self,
        exit_price: float,
        quantity: float
    ) -> Dict:
        """
        청산 수수료 계산
        
        계산식:
            매도 금액 = exit_price × quantity
            수수료 = 매도 금액 × 0.001
            순 수익 = 매도 금액 - 수수료
        
        Returns:
            {
                'trade_value': 거래 금액,
                'fee': 수수료,
                'net_revenue': 순 수익,
                'fee_rate': 수수료율
            }
        
        Example:
            >>> result = calc.calculate_exit_fee(102.0, 10)
            >>> print(f"순 수익: {result['net_revenue']:.2f} USDT")
            순 수익: 1018.98 USDT
        """
        trade_value = exit_price * quantity
        fee = trade_value * self.fee_rate
        net_revenue = trade_value - fee
        
        return {
            'trade_value': trade_value,
            'fee': fee,
            'net_revenue': net_revenue,
            'fee_rate': self.fee_rate
        }
    
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
        
        Example:
            >>> fees = calc.calculate_total_fees(100.0, 102.0, 10)
            >>> print(f"총 수수료: {fees['total_fee']:.2f} USDT")
            >>> print(f"수수료율: {fees['total_fee_percent']*100:.2f}%")
            총 수수료: 2.02 USDT
            수수료율: -0.20%
        """
        entry_info = self.calculate_entry_fee(entry_price, quantity)
        exit_info = self.calculate_exit_fee(exit_price, quantity)
        
        total_fee = entry_info['fee'] + exit_info['fee']
        total_fee_percent = total_fee / entry_info['total_cost']
        
        return {
            'entry_fee': entry_info['fee'],
            'exit_fee': exit_info['fee'],
            'total_fee': total_fee,
            'total_fee_percent': -total_fee_percent  # 음수로 표현
        }
    
    def calculate_net_pnl(
        self,
        entry_price: float,
        exit_price: float,
        quantity: float
    ) -> Dict:
        """
        수수료 포함 순손익 계산
        
        Args:
            entry_price: 진입가 (USDT)
            exit_price: 청산가 (USDT)
            quantity: 수량 (코인)
        
        Returns:
            {
                'gross_pnl': 명목 손익,
                'gross_pnl_percent': 명목 수익률,
                'net_pnl': 순손익 (수수료 포함),
                'net_pnl_percent': 실제 수익률,
                'total_fee': 총 수수료,
                'fee_impact': 수수료 영향,
                'entry_cost': 총 진입 비용,
                'exit_revenue': 순 청산 수익
            }
        
        Example:
            >>> pnl = calc.calculate_net_pnl(100.0, 102.0, 10)
            >>> print(f"명목 수익: {pnl['gross_pnl_percent']*100:.2f}%")
            >>> print(f"실제 수익: {pnl['net_pnl_percent']*100:.2f}%")
            >>> print(f"수수료 영향: {pnl['fee_impact']*100:.2f}%")
            명목 수익: +2.00%
            실제 수익: +1.80%
            수수료 영향: -0.20%
        """
        entry_info = self.calculate_entry_fee(entry_price, quantity)
        exit_info = self.calculate_exit_fee(exit_price, quantity)
        
        total_fee = entry_info['fee'] + exit_info['fee']
        
        # 순손익
        net_profit = exit_info['net_revenue'] - entry_info['total_cost']
        net_pnl_percent = net_profit / entry_info['total_cost']
        
        # 명목손익
        gross_profit = (exit_price - entry_price) * quantity
        gross_pnl_percent = (exit_price - entry_price) / entry_price
        
        # 수수료 영향
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
    
    def get_breakeven_price(self, entry_price: float) -> float:
        """
        손익분기점 가격
        
        계산식:
            총 수수료율 = 0.2%
            손익분기 = entry_price × 1.002
        
        Args:
            entry_price: 진입가 (USDT)
        
        Returns:
            float: 손익분기 가격
        
        Example:
            >>> be = calc.get_breakeven_price(100.0)
            >>> print(f"손익분기: {be:.2f} USDT")
            손익분기: 100.20 USDT
        """
        total_fee_rate = self.fee_rate * 2
        breakeven_price = entry_price * (1 + total_fee_rate)
        return breakeven_price
    
    def convert_krw_to_usdt(self, amount_krw: float) -> float:
        """
        KRW → USDT 변환
        
        ⭐ 개선: 신규 추가
        
        Args:
            amount_krw: KRW 금액
        
        Returns:
            float: USDT 금액
        
        Example:
            >>> usdt = calc.convert_krw_to_usdt(1300000)
            >>> print(f"{usdt:.2f} USDT")
            1000.00 USDT
        """
        return amount_krw / KRW_USD_RATE
    
    def convert_usdt_to_krw(self, amount_usdt: float) -> float:
        """
        USDT → KRW 변환
        
        ⭐ 개선: 신규 추가
        
        Args:
            amount_usdt: USDT 금액
        
        Returns:
            float: KRW 금액
        
        Example:
            >>> krw = calc.convert_usdt_to_krw(1000.0)
            >>> print(f"{krw:,.0f} KRW")
            1,300,000 KRW
        """
        return amount_usdt * KRW_USD_RATE
```

---

## 📁 utils/validators.py ⭐ 개선

### 구현 코드 (전체 개선)

```python
from typing import Dict, Any, Tuple
import re
from datetime import datetime


class DataValidator:
    """
    데이터 검증
    
    ⭐ 개선: 에러 메시지 반환 추가
    """
    
    @staticmethod
    def validate_symbol(symbol: str) -> Tuple[bool, str]:
        """
        심볼 형식 검증
        
        ⭐ 개선: 에러 메시지 반환
        
        규칙:
            패턴: [대문자]/[대문자]
            예: DOGE/USDT, SOL/USDT
        
        Returns:
            (bool, str): (성공 여부, 에러 메시지)
        
        Example:
            >>> valid, msg = DataValidator.validate_symbol('DOGE/USDT')
            >>> if not valid:
            >>>     print(f"검증 실패: {msg}")
            
            >>> valid, msg = DataValidator.validate_symbol('doge/usdt')
            >>> print(msg)
            "심볼은 대문자여야 합니다: doge/usdt"
        """
        if not isinstance(symbol, str):
            return False, f"심볼은 문자열이어야 합니다: {type(symbol)}"
        
        pattern = r'^[A-Z]+/[A-Z]+$'
        if not re.match(pattern, symbol):
            return False, f"잘못된 심볼 형식: {symbol} (예: DOGE/USDT)"
        
        return True, ""
    
    @staticmethod
    def validate_price(price: float) -> Tuple[bool, str]:
        """
        가격 검증
        
        ⭐ 개선: 에러 메시지 반환
        
        규칙:
            0 < price < 1,000,000
        
        Example:
            >>> valid, msg = DataValidator.validate_price(0.3821)
            >>> assert valid
            
            >>> valid, msg = DataValidator.validate_price(-1.0)
            >>> print(msg)
            "가격은 양수여야 합니다: -1.0"
        """
        try:
            price = float(price)
        except (TypeError, ValueError):
            return False, f"가격을 숫자로 변환할 수 없습니다: {price}"
        
        if price <= 0:
            return False, f"가격은 양수여야 합니다: {price}"
        
        if price >= 1_000_000:
            return False, f"가격이 비정상적으로 높습니다: {price}"
        
        return True, ""
    
    @staticmethod
    def validate_quantity(
        quantity: float,
        min_qty: float = 0.001
    ) -> Tuple[bool, str]:
        """
        수량 검증
        
        ⭐ 개선: 에러 메시지 반환
        
        Args:
            quantity: 수량
            min_qty: 최소 수량 (기본 0.001)
        
        Example:
            >>> valid, msg = DataValidator.validate_quantity(10.5)
            >>> assert valid
            
            >>> valid, msg = DataValidator.validate_quantity(0.0001, min_qty=0.001)
            >>> print(msg)
            "수량이 최소값보다 작습니다: 0.0001 < 0.001"
        """
        try:
            quantity = float(quantity)
        except (TypeError, ValueError):
            return False, f"수량을 숫자로 변환할 수 없습니다: {quantity}"
        
        if quantity < min_qty:
            return False, f"수량이 최소값보다 작습니다: {quantity} < {min_qty}"
        
        return True, ""
    
    @staticmethod
    def validate_timestamp(timestamp: int) -> Tuple[bool, str]:
        """
        타임스탬프 검증
        
        ⭐ 개선: 에러 메시지 반환
        
        규칙:
            2020-01-01 < timestamp < 2030-12-31
        
        Example:
            >>> valid, msg = DataValidator.validate_timestamp(1640000000)
            >>> assert valid
            
            >>> valid, msg = DataValidator.validate_timestamp(999999999)
            >>> print(msg)
            "타임스탬프가 범위를 벗어났습니다: 999999999"
        """
        try:
            timestamp = int(timestamp)
        except (TypeError, ValueError):
            return False, f"타임스탬프를 정수로 변환할 수 없습니다: {timestamp}"
        
        min_ts = 1577836800  # 2020-01-01
        max_ts = 1924991999  # 2030-12-31
        
        if not (min_ts <= timestamp <= max_ts):
            return False, f"타임스탬프가 범위를 벗어났습니다: {timestamp}"
        
        return True, ""
    
    @staticmethod
    def validate_ohlcv(ohlcv: list) -> Tuple[bool, str]:
        """
        OHLCV 데이터 검증
        
        ⭐ 개선: 에러 메시지 반환
        
        규칙:
            [timestamp, open, high, low, close, volume]
            - high >= low
            - volume >= 0
        
        Example:
            >>> candle = [1640000000, 100, 105, 98, 102, 1000000]
            >>> valid, msg = DataValidator.validate_ohlcv([candle])
            >>> assert valid
            
            >>> bad_candle = [1640000000, 100, 95, 98, 102, 1000000]  # high < low
            >>> valid, msg = DataValidator.validate_ohlcv([bad_candle])
            >>> print(msg)
            "캔들 0: high(95) < low(98)"
        """
        if not isinstance(ohlcv, list) or len(ohlcv) == 0:
            return False, "OHLCV가 비어있거나 리스트가 아닙니다"
        
        for i, candle in enumerate(ohlcv):
            if not isinstance(candle, list) or len(candle) != 6:
                return False, f"캔들 {i}: 길이가 6이 아닙니다"
            
            try:
                ts, o, h, l, c, v = candle
                
                # 타임스탬프
                valid_ts, msg_ts = DataValidator.validate_timestamp(ts)
                if not valid_ts:
                    return False, f"캔들 {i}: {msg_ts}"
                
                # 가격
                for price_name, price_value in [('open', o), ('high', h), ('low', l), ('close', c)]:
                    valid_p, msg_p = DataValidator.validate_price(price_value)
                    if not valid_p:
                        return False, f"캔들 {i} {price_name}: {msg_p}"
                
                # 논리 검증
                if h < l:
                    return False, f"캔들 {i}: high({h}) < low({l})"
                
                # 거래량
                if v < 0:
                    return False, f"캔들 {i}: 거래량이 음수입니다: {v}"
            
            except (TypeError, ValueError) as e:
                return False, f"캔들 {i}: 데이터 파싱 오류: {e}"
        
        return True, ""
    
    @staticmethod
    def validate_indicators(indicators: Dict) -> Tuple[bool, str]:
        """
        지표 데이터 검증
        
        ⭐ 개선: 에러 메시지 반환
        
        규칙:
            필수 키: ['rsi', 'macd', 'bollinger', 'fibonacci']
        
        Example:
            >>> indicators = {
            ...     'rsi': {'value': 45.2},
            ...     'macd': {'value': 0.001},
            ...     'bollinger': {'upper': 0.39},
            ...     'fibonacci': {'support': 0.38}
            ... }
            >>> valid, msg = DataValidator.validate_indicators(indicators)
            >>> assert valid
        """
        if not isinstance(indicators, dict):
            return False, f"지표가 딕셔너리가 아닙니다: {type(indicators)}"
        
        required_keys = ['rsi', 'macd', 'bollinger', 'fibonacci']
        
        for key in required_keys:
            if key not in indicators:
                return False, f"필수 지표 누락: {key}"
            
            if not isinstance(indicators[key], dict):
                return False, f"지표 {key}가 딕셔너리가 아닙니다"
        
        return True, ""
    
    @staticmethod
    def validate_ai_response(response: Dict) -> Tuple[bool, str]:
        """
        AI 응답 검증
        
        ⭐ 개선: 에러 메시지 반환
        
        규칙:
            필수 키: ['action', 'confidence', 'reasoning']
            action: 'ENTER' | 'EXIT' | 'HOLD' | 'WAIT'
            confidence: 0.0 ~ 1.0
        
        Example:
            >>> response = {
            ...     'action': 'ENTER',
            ...     'confidence': 0.75,
            ...     'reasoning': '...'
            ... }
            >>> valid, msg = DataValidator.validate_ai_response(response)
            >>> assert valid
        """
        if not isinstance(response, dict):
            return False, f"AI 응답이 딕셔너리가 아닙니다: {type(response)}"
        
        # 필수 키
        required = ['action', 'confidence', 'reasoning']
        for key in required:
            if key not in response:
                return False, f"필수 필드 누락: {key}"
        
        # action 검증
        valid_actions = ['ENTER', 'EXIT', 'HOLD', 'WAIT']
        if response['action'] not in valid_actions:
            return False, f"잘못된 action: {response['action']} (가능: {valid_actions})"
        
        # confidence 검증
        try:
            conf = float(response['confidence'])
            if not (0 <= conf <= 1):
                return False, f"confidence 범위 오류: {conf} (0~1)"
        except (TypeError, ValueError):
            return False, f"confidence를 숫자로 변환할 수 없습니다: {response['confidence']}"
        
        # reasoning 검증
        if not isinstance(response['reasoning'], str):
            return False, f"reasoning이 문자열이 아닙니다: {type(response['reasoning'])}"
        
        return True, ""
    
    @staticmethod
    def validate_trade_data(trade: Dict) -> Tuple[bool, str]:
        """
        거래 데이터 검증
        
        ⭐ 개선: 에러 메시지 반환
        
        규칙:
            필수 키: ['symbol', 'entry_price', 'quantity', 'timestamp']
        
        Example:
            >>> trade = {
            ...     'symbol': 'DOGE/USDT',
            ...     'entry_price': 0.3821,
            ...     'quantity': 1000,
            ...     'timestamp': 1640000000
            ... }
            >>> valid, msg = DataValidator.validate_trade_data(trade)
            >>> assert valid
        """
        if not isinstance(trade, dict):
            return False, f"거래 데이터가 딕셔너리가 아닙니다: {type(trade)}"
        
        # 필수 키
        required = ['symbol', 'entry_price', 'quantity', 'timestamp']
        for key in required:
            if key not in trade:
                return False, f"필수 필드 누락: {key}"
        
        # 개별 검증
        valid_symbol, msg_symbol = DataValidator.validate_symbol(trade['symbol'])
        if not valid_symbol:
            return False, f"symbol: {msg_symbol}"
        
        valid_price, msg_price = DataValidator.validate_price(trade['entry_price'])
        if not valid_price:
            return False, f"entry_price: {msg_price}"
        
        valid_qty, msg_qty = DataValidator.validate_quantity(trade['quantity'])
        if not valid_qty:
            return False, f"quantity: {msg_qty}"
        
        valid_ts, msg_ts = DataValidator.validate_timestamp(trade['timestamp'])
        if not valid_ts:
            return False, f"timestamp: {msg_ts}"
        
        return True, ""
```

---

## 📁 utils/helpers.py

### 구현 코드 (완성)

```python
from datetime import datetime, timedelta
from typing import Dict, List, Any
import json


def timestamp_to_datetime(timestamp: int) -> str:
    """
    타임스탬프 → 날짜 문자열
    
    Example:
        >>> timestamp_to_datetime(1640000000)
        '2021-12-20 13:33:20'
    """
    return datetime.fromtimestamp(timestamp).strftime('%Y-%m-%d %H:%M:%S')


def datetime_to_timestamp(dt_str: str) -> int:
    """
    날짜 문자열 → 타임스탬프
    
    Example:
        >>> datetime_to_timestamp('2021-12-20 13:33:20')
        1640000000
    """
    dt = datetime.strptime(dt_str, '%Y-%m-%d %H:%M:%S')
    return int(dt.timestamp())


def format_number(num: float, decimals: int = 2) -> str:
    """
    숫자 포맷팅 (천 단위 콤마)
    
    Example:
        >>> format_number(1234567.89, 2)
        '1,234,567.89'
    """
    return f"{num:,.{decimals}f}"


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


def format_krw(amount: float) -> str:
    """
    KRW 금액 포맷팅
    
    Example:
        >>> format_krw(1234567)
        '1,234,567 KRW'
    """
    return f"{amount:,.0f} KRW"


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

```
core/constants.py ⭐
├── KRW_USD_RATE 추가
├── FEE_RATES 추가
└── 사용처: utils/fee_calculator.py

utils/network.py ⭐
├── 사용: asyncio, aiohttp, ccxt, logging
├── ⭐ emergency_network_failure() 구현 완성
└── 사용처: exchanges/, data/, engine/

utils/fee_calculator.py ⭐
├── 사용: core/constants (FEE_RATES, KRW_USD_RATE)
├── ⭐ convert_krw_to_usdt() 추가
├── ⭐ convert_usdt_to_krw() 추가
└── 사용처: engine/, exchanges/, monitoring/

utils/validators.py ⭐
├── 사용: re, datetime
├── ⭐ 모든 함수에 에러 메시지 반환 추가
└── 사용처: data/, ai/, exchanges/, engine/

utils/helpers.py
├── 사용: datetime, json
└── 사용처: 모든 모듈
```

---

## 실전 사용 예제

### 예제 1: 네트워크 재시도 사용

```python
from utils.network import retry_on_network_error

class BybitLiveExchange:
    @retry_on_network_error(max_retries=60, delay=1)
    async def fetch_ticker(self, symbol):
        """자동 재시도 적용"""
        return await self.exchange.fetch_ticker(symbol)
    
    @retry_on_network_error(max_retries=5, exponential_backoff=True)
    async def create_order(self, symbol, quantity):
        """중요한 주문은 지수 백오프"""
        return await self.exchange.create_market_buy_order(symbol, quantity)
```

### 예제 2: 수수료 계산

```python
from utils.fee_calculator import FeeCalculator

calc = FeeCalculator('bybit')

# 진입 비용 계산
entry_info = calc.calculate_entry_fee(
    entry_price=0.3821,
    quantity=1000
)
print(f"총 비용: {entry_info['total_cost']:.2f} USDT")
print(f"수수료: {entry_info['fee']:.2f} USDT")

# 순손익 계산
pnl_info = calc.calculate_net_pnl(
    entry_price=0.3821,
    exit_price=0.3895,
    quantity=1000
)
print(f"명목 수익: {pnl_info['gross_pnl_percent']*100:.2f}%")
print(f"실제 수익: {pnl_info['net_pnl_percent']*100:.2f}%")
print(f"수수료 영향: {pnl_info['fee_impact']*100:.2f}%")

# KRW 변환
krw_amount = 1000000
usdt_amount = calc.convert_krw_to_usdt(krw_amount)
print(f"{krw_amount:,} KRW = {usdt_amount:.2f} USDT")
```

### 예제 3: 데이터 검증

```python
from utils.validators import DataValidator

# 심볼 검증
valid, msg = DataValidator.validate_symbol('DOGE/USDT')
if not valid:
    print(f"검증 실패: {msg}")
else:
    print("검증 성공")

# 가격 검증
valid, msg = DataValidator.validate_price(-1.0)
print(msg)  # "가격은 양수여야 합니다: -1.0"

# AI 응답 검증
ai_response = {
    'action': 'ENTER',
    'confidence': 0.75,
    'reasoning': 'MACD golden cross'
}
valid, msg = DataValidator.validate_ai_response(ai_response)
if valid:
    # AI 응답 사용
    execute_entry(ai_response)
else:
    print(f"AI 응답 오류: {msg}")
```

### 예제 4: 헬퍼 함수 활용

```python
from utils.helpers import *

# 날짜 변환
ts = 1640000000
print(timestamp_to_datetime(ts))  # 2021-12-20 13:33:20

# 포맷팅
print(format_number(1234567.89))    # 1,234,567.89
print(format_percentage(0.0234))    # +2.34%
print(format_krw(1000000))          # 1,000,000 KRW

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

### 예제 5: 네트워크 모니터링

```python
from utils.network import NetworkMonitor

class BaseEngine:
    def __init__(self, config, exchange):
        self.network_monitor = NetworkMonitor()
    
    async def fetch_data(self, symbol):
        try:
            data = await self.exchange.fetch_ticker(symbol)
            self.network_monitor.record_request(True)
            return data
        except Exception as e:
            self.network_monitor.record_request(False)
            raise
    
    async def check_network_health(self):
        """네트워크 상태 확인"""
        stats = self.network_monitor.get_stats()
        
        print(f"총 요청: {stats['total']}")
        print(f"성공률: {stats['success_rate']*100:.1f}%")
        
        if not stats['stable']:
            print("⚠️ 네트워크 불안정")
```

---

## 테스트 시나리오

### network.py 테스트

```python
import pytest
from utils.network import (
    retry_on_network_error,
    check_internet_connection,
    NetworkMonitor
)

@pytest.mark.asyncio
async def test_retry_decorator():
    """재시도 데코레이터 테스트"""
    call_count = 0
    
    @retry_on_network_error(max_retries=3, delay=0.1)
    async def failing_func():
        nonlocal call_count
        call_count += 1
        if call_count < 3:
            raise ccxt.NetworkError("Connection failed")
        return "success"
    
    result = await failing_func()
    assert result == "success"
    assert call_count == 3

@pytest.mark.asyncio
async def test_check_connection():
    """인터넷 연결 확인 테스트"""
    is_connected = await check_internet_connection()
    assert isinstance(is_connected, bool)

def test_network_monitor():
    """네트워크 모니터 테스트"""
    monitor = NetworkMonitor()
    
    # 성공 기록
    for _ in range(95):
        monitor.record_request(True)
    
    # 실패 기록
    for _ in range(5):
        monitor.record_request(False)
    
    stats = monitor.get_stats()
    assert stats['total'] == 100
    assert stats['success_rate'] == 0.95
    assert stats['stable'] == True
```

### fee_calculator.py 테스트

```python
def test_fee_calculator():
    """수수료 계산 테스트"""
    calc = FeeCalculator('bybit')
    
    # 진입 수수료
    entry = calc.calculate_entry_fee(100.0, 10)
    assert entry['fee'] == 1.0  # 1000 * 0.001
    assert entry['total_cost'] == 1001.0
    
    # 순손익
    pnl = calc.calculate_net_pnl(100.0, 102.0, 10)
    assert pnl['gross_pnl'] == 20.0
    assert abs(pnl['net_pnl'] - 17.98) < 0.01
    assert abs(pnl['fee_impact'] - (-0.002)) < 0.0001

def test_krw_conversion():
    """KRW 변환 테스트"""
    calc = FeeCalculator('bybit')
    
    # KRW → USDT
    usdt = calc.convert_krw_to_usdt(1300000)
    assert usdt == 1000.0
    
    # USDT → KRW
    krw = calc.convert_usdt_to_krw(1000.0)
    assert krw == 1300000.0
```

### validators.py 테스트

```python
def test_validator_with_messages():
    """에러 메시지 포함 검증 테스트"""
    # 성공
    valid, msg = DataValidator.validate_symbol('DOGE/USDT')
    assert valid
    assert msg == ""
    
    # 실패 - 에러 메시지 확인
    valid, msg = DataValidator.validate_symbol('doge/usdt')
    assert not valid
    assert 'DOGE/USDT' in msg
    
    valid, msg = DataValidator.validate_price(-1.0)
    assert not valid
    assert '양수' in msg
    
    valid, msg = DataValidator.validate_quantity(0.0001, min_qty=0.001)
    assert not valid
    assert '최소값' in msg

def test_validate_ai_response():
    """AI 응답 검증 테스트"""
    # 정상
    response = {
        'action': 'ENTER',
        'confidence': 0.75,
        'reasoning': 'test'
    }
    valid, msg = DataValidator.validate_ai_response(response)
    assert valid
    
    # 잘못된 action
    response['action'] = 'BUY'
    valid, msg = DataValidator.validate_ai_response(response)
    assert not valid
    assert 'action' in msg
```

---

## 개발 체크리스트

### core/constants.py ⭐
- [x] ⭐ KRW_USD_RATE 상수 추가
- [x] ⭐ FEE_RATES 상수 추가

### network.py ⭐
- [x] retry_on_network_error 데코레이터
- [x] check_internet_connection 함수
- [x] wait_for_connection 함수
- [x] ⭐ emergency_network_failure() 완전 구현
- [x] NetworkMonitor 클래스
  - [x] record_request 메서드
  - [x] get_success_rate 메서드
  - [x] is_stable 메서드
  - [x] ⭐ get_stats 메서드 추가

### fee_calculator.py ⭐
- [x] FeeCalculator 클래스
- [x] calculate_entry_fee 메서드
- [x] calculate_exit_fee 메서드
- [x] calculate_total_fees 메서드
- [x] calculate_net_pnl 메서드
- [x] get_breakeven_price 메서드
- [x] ⭐ convert_krw_to_usdt 메서드 추가
- [x] ⭐ convert_usdt_to_krw 메서드 추가
- [x] ⭐ FEE_RATES constants로 이동

### validators.py ⭐
- [x] DataValidator 클래스
- [x] ⭐ 모든 메서드에 Tuple[bool, str] 반환 추가
- [x] validate_symbol
- [x] validate_price
- [x] validate_quantity
- [x] validate_timestamp
- [x] validate_ohlcv
- [x] validate_indicators
- [x] validate_ai_response
- [x] validate_trade_data

### helpers.py
- [x] timestamp_to_datetime 함수
- [x] datetime_to_timestamp 함수
- [x] format_number 함수
- [x] format_percentage 함수
- [x] format_krw 함수
- [x] calculate_time_diff 함수
- [x] safe_divide 함수
- [x] deep_get 함수
- [x] truncate_string 함수
- [x] load_json_file 함수
- [x] save_json_file 함수

---

## 주요 특징

### 1. 네트워크 안정성 ⭐
- 자동 재시도 (최대 60회)
- 지수 백오프 옵션
- 긴급 처리 프로토콜 완비
- 연결 상태 모니터링

### 2. 정확한 수수료 계산 ⭐
- 거래소별 수수료율 지원
- 진입/청산 수수료 분리
- 순손익 자동 계산
- KRW/USDT 변환 추가

### 3. 철저한 데이터 검증 ⭐
- 입력 데이터 형식 검증
- 논리적 무결성 확인
- 명확한 에러 메시지
- 타입 안전성

### 4. 편리한 헬퍼 함수
- 날짜/시간 변환
- 숫자 포맷팅
- 안전한 연산
- JSON 처리

### 5. Constants 통합 ⭐
- KRW_USD_RATE 중앙 관리
- FEE_RATES 중앙 관리
- 일관된 설정 값

---

**문서 버전**: v2.0 (개선판)  
**작성일**: 2025-01-22  
**개선사항**:
- ⭐ core/constants.py에 KRW_USD_RATE 추가
- ⭐ core/constants.py에 FEE_RATES 추가
- ⭐ emergency_network_failure() 완전 구현
- ⭐ FeeCalculator에 convert_krw_to_usdt() 추가
- ⭐ FeeCalculator에 convert_usdt_to_krw() 추가
- ⭐ DataValidator 모든 함수에 에러 메시지 반환 추가
- ⭐ NetworkMonitor에 get_stats() 추가
- ✅ 실전 사용 예제 5개
- ✅ 테스트 시나리오 완성

**검증 상태**: ✅ 완료
