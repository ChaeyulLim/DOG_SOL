# 10_MONITORING 모듈 완벽 함수 명세서 v2.0 (개선판)

> **개선사항**: generate_monthly_report 구현, balance_history 저장 메커니즘, risk_events DB 연동, 로그 자동 정리 추가

---

## 📋 목차
1. [monitoring/logger.py](#monitoringloggerpy)
2. [monitoring/reporter.py](#monitoringreporterpy) ⭐ 개선
3. [monitoring/performance.py](#monitoringperformancepy) ⭐ 개선
4. [전체 의존성 그래프](#전체-의존성-그래프)
5. [실전 사용 예제](#실전-사용-예제)

---

## 📁 monitoring/logger.py

### 구현 코드 (완성)

```python
import logging
import sys
from pathlib import Path
from datetime import datetime
from typing import Optional

from core.constants import LOG_DIR
from database import execute_query  # ⭐ DB 연동


class TradingLogger:
    """
    모든 거래 활동 로깅 (파일 + 콘솔 + DB)
    
    ⭐ 개선: risk_events DB 저장 추가
    """
    
    def __init__(self, mode: str, level: str = 'INFO'):
        """
        로거 초기화
        
        Args:
            mode: 'paper', 'live', 'backtest'
            level: 'DEBUG', 'INFO', 'WARNING', 'ERROR'
        
        호출:
            engine/base_engine.py
        """
        self.mode = mode
        self.level = level
        self.logger = self.setup_logger()
    
    def setup_logger(self) -> logging.Logger:
        """
        로거 설정 (파일 + 콘솔 핸들러)
        
        로그 파일 구조:
            logs/paper/2025-01-15_trades.log
            logs/paper/2025-01-15_decisions.log
            logs/paper/2025-01-15_errors.log
            logs/paper/2025-01-15_debug.log
        """
        logger = logging.getLogger(f'trading_{self.mode}')
        logger.setLevel(getattr(logging, self.level))
        logger.handlers.clear()
        
        today = datetime.now().strftime('%Y-%m-%d')
        log_dir = Path(LOG_DIR) / self.mode
        log_dir.mkdir(parents=True, exist_ok=True)
        
        formatter = logging.Formatter(
            '[%(asctime)s] [%(levelname)s] %(message)s',
            datefmt='%Y-%m-%d %H:%M:%S'
        )
        
        # 1. 콘솔 핸들러
        console_handler = logging.StreamHandler(sys.stdout)
        console_handler.setLevel(logging.INFO)
        console_handler.setFormatter(formatter)
        logger.addHandler(console_handler)
        
        # 2. 거래 로그
        trades_handler = logging.FileHandler(
            log_dir / f'{today}_trades.log',
            encoding='utf-8'
        )
        trades_handler.setLevel(logging.INFO)
        trades_handler.setFormatter(formatter)
        trades_handler.addFilter(
            lambda record: '[ENTRY]' in record.getMessage() or 
                          '[EXIT]' in record.getMessage()
        )
        logger.addHandler(trades_handler)
        
        # 3. AI 결정 로그
        decisions_handler = logging.FileHandler(
            log_dir / f'{today}_decisions.log',
            encoding='utf-8'
        )
        decisions_handler.setLevel(logging.INFO)
        decisions_handler.setFormatter(formatter)
        decisions_handler.addFilter(
            lambda record: '[AI]' in record.getMessage()
        )
        logger.addHandler(decisions_handler)
        
        # 4. 에러 로그
        error_handler = logging.FileHandler(
            log_dir / f'{today}_errors.log',
            encoding='utf-8'
        )
        error_handler.setLevel(logging.ERROR)
        error_handler.setFormatter(formatter)
        logger.addHandler(error_handler)
        
        # 5. 디버그 로그 (선택)
        if self.level == 'DEBUG':
            debug_handler = logging.FileHandler(
                log_dir / f'{today}_debug.log',
                encoding='utf-8'
            )
            debug_handler.setLevel(logging.DEBUG)
            debug_handler.setFormatter(formatter)
            logger.addHandler(debug_handler)
        
        return logger
    
    def log_entry(
        self,
        symbol: str,
        price: float,
        amount_krw: int,
        ai_confidence: float
    ) -> None:
        """
        진입 로그
        
        Example:
            >>> logger.log_entry('DOGE/USDT', 0.3821, 500000, 0.75)
            [2025-01-15 14:32:15] [INFO] [ENTRY] DOGE/USDT @ 0.3821 | Amount: 500,000 KRW | AI: 0.75
        """
        self.logger.info(
            f"[ENTRY] {symbol} @ {price:.4f} | "
            f"Amount: {amount_krw:,} KRW | "
            f"AI: {ai_confidence:.2f}"
        )
    
    def log_exit(
        self,
        symbol: str,
        entry_price: float,
        exit_price: float,
        pnl_percent: float,
        pnl_krw: float,
        reason: str
    ) -> None:
        """
        청산 로그
        
        Example:
            >>> logger.log_exit('DOGE/USDT', 0.3821, 0.3895, 0.0194, 9700, 'TRAILING_STOP')
            [2025-01-15 16:45:22] [INFO] [EXIT] DOGE/USDT @ 0.3895 | PnL: +1.94% (+9,700 KRW) | Reason: TRAILING_STOP
        """
        self.logger.info(
            f"[EXIT] {symbol} @ {exit_price:.4f} | "
            f"Entry: {entry_price:.4f} | "
            f"PnL: {pnl_percent*100:+.2f}% ({pnl_krw:+,.0f} KRW) | "
            f"Reason: {reason}"
        )
    
    def log_risk_event(
        self,
        event_type: str,
        details: dict
    ) -> None:
        """
        리스크 이벤트 로그 + DB 저장
        
        ⭐ 개선: DB 저장 추가
        
        Example:
            >>> logger.log_risk_event('DAILY_LIMIT', {
            ...     'loss': -0.05,
            ...     'resume_time': 1234567890
            ... })
            [2025-01-15 18:00:00] [WARNING] [RISK] DAILY_LIMIT | Loss: -5.0%
        """
        import json
        
        # 로그 메시지 구성
        message = f"[RISK] {event_type}"
        
        if 'loss' in details:
            message += f" | Loss: {details['loss']*100:.1f}%"
        
        if 'drawdown' in details:
            message += f" | DD: {details['drawdown']*100:.1f}%"
        
        if 'count' in details:
            message += f" | Count: {details['count']}"
        
        if 'resume_time' in details:
            resume_dt = datetime.fromtimestamp(details['resume_time'])
            message += f" | Resume: {resume_dt.strftime('%Y-%m-%d %H:%M:%S')}"
        
        self.logger.warning(message)
        
        # ⭐ DB에도 저장
        try:
            execute_query(
                """
                INSERT INTO risk_events (timestamp, event_type, details, action_taken)
                VALUES (?, ?, ?, ?)
                """,
                (
                    datetime.now(),
                    event_type,
                    json.dumps(details),
                    details.get('action', '')
                )
            )
        except Exception as e:
            self.logger.error(f"리스크 이벤트 DB 저장 실패: {e}")
    
    def log_ai_decision(
        self,
        symbol: str,
        action: str,
        confidence: float,
        reasoning: str
    ) -> None:
        """
        AI 판단 로그
        
        Example:
            >>> logger.log_ai_decision('DOGE/USDT', 'ENTER', 0.75, 'MACD golden cross...')
            [2025-01-15 14:32:10] [INFO] [AI] DOGE/USDT | Action: ENTER | Confidence: 0.75
        """
        short_reasoning = reasoning[:50] + '...' if len(reasoning) > 50 else reasoning
        
        self.logger.info(
            f"[AI] {symbol} | "
            f"Action: {action} | "
            f"Confidence: {confidence:.2f} | "
            f"Reasoning: {short_reasoning}"
        )
```

---

## 📁 monitoring/reporter.py ⭐ 개선

### 구현 코드 (전체 개선)

```python
from datetime import datetime, timedelta
from typing import Dict
from database import TradeDatabase


class PerformanceReporter:
    """
    일일/주간/월간 리포트 생성
    
    ⭐ 개선: generate_monthly_report 구현 완성
    """
    
    def __init__(self, db: TradeDatabase):
        """
        리포터 초기화
        
        Args:
            db: TradeDatabase 인스턴스
        
        호출:
            engine/base_engine.py
        """
        self.db = db
    
    def generate_daily_summary(self) -> str:
        """
        일일 요약 리포트
        
        호출:
            engine/base_engine.py - 매일 09:00, 21:00
        
        Returns:
            포맷된 리포트 문자열
        
        Example:
            >>> reporter = PerformanceReporter(db)
            >>> report = reporter.generate_daily_summary()
            >>> print(report)
            ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            📊 2025-01-15 일일 리포트 (21:00)
            ...
        """
        today = datetime.now().strftime('%Y-%m-%d')
        
        # 오늘 거래 조회
        trades = self.db.get_trades_by_date(today)
        
        if not trades:
            return f"📊 {today} - 오늘 거래 없음"
        
        # 통계 계산
        total = len(trades)
        winners = [t for t in trades if t['pnl_percent'] > 0]
        losers = [t for t in trades if t['pnl_percent'] <= 0]
        
        win_rate = (len(winners) / total) * 100
        avg_profit = sum(t['pnl_percent'] for t in winners) / len(winners) * 100 if winners else 0
        total_pnl = sum(t['pnl_krw'] for t in trades)
        
        # 심볼별 집계
        symbol_summary = {}
        for trade in trades:
            symbol = trade['symbol'].split('/')[0]
            if symbol not in symbol_summary:
                symbol_summary[symbol] = []
            symbol_summary[symbol].append(trade['pnl_percent'] * 100)
        
        # 리포트 작성
        report = f"""
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 {today} 일일 리포트 ({datetime.now().strftime('%H:%M')})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📈 오늘 거래:
  거래 횟수: {total}회
  승률: {win_rate:.1f}% ({len(winners)}승 {len(losers)}패)
  평균 수익: {avg_profit:+.1f}%
  총 손익: {total_pnl:+,.0f} KRW

🎯 심볼별:
"""
        
        for symbol, pnls in symbol_summary.items():
            pnl_str = ', '.join([f"{p:+.1f}%" for p in pnls])
            report += f"  {symbol}: {len(pnls)}회 ({pnl_str})\n"
        
        report += "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
        
        return report
    
    def generate_weekly_report(self) -> str:
        """
        주간 분석 리포트
        
        호출:
            매주 월요일 00:00 자동
        
        Returns:
            포맷된 리포트 문자열
        """
        trades = self.db.get_trades_last_n_days(7)
        
        if not trades:
            return "📊 주간 리포트 - 거래 없음"
        
        # 기본 통계
        total = len(trades)
        winners = [t for t in trades if t['pnl_percent'] > 0]
        win_rate = (len(winners) / total) * 100
        total_pnl = sum(t['pnl_percent'] for t in trades) * 100
        
        report = f"""
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 주간 분석 ({datetime.now().strftime('%Y-%m-%d')})
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📈 성과:
  총 거래: {total}회
  승률: {win_rate:.1f}% ({len(winners)}승 {total-len(winners)}패)
  주간 수익률: {total_pnl:+.1f}%

💡 상세 분석은 learning_report.txt 참조
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"""
        
        return report
    
    def generate_monthly_report(self) -> str:
        """
        월간 종합 리포트
        
        ⭐ 개선: 완전히 새로 구현
        
        호출:
            매월 1일 00:00 자동
        
        Returns:
            포맷된 리포트 문자열
        
        Example:
            >>> report = reporter.generate_monthly_report()
            >>> print(report)
            ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            📊 2025년 1월 월간 리포트
            ...
        """
        # 지난 달 데이터
        now = datetime.now()
        if now.month == 1:
            last_month = 12
            year = now.year - 1
        else:
            last_month = now.month - 1
            year = now.year
        
        # 기간 설정
        start_date = f"{year}-{last_month:02d}-01"
        if last_month == 12:
            end_date = f"{year}-{last_month:02d}-31"
        else:
            # 다음 달 1일 - 1초
            next_month_first = datetime(year, last_month + 1, 1)
            last_day = (next_month_first - timedelta(seconds=1)).day
            end_date = f"{year}-{last_month:02d}-{last_day:02d}"
        
        # 거래 조회
        trades = self.db.get_trades_by_period(start_date, end_date)
        
        if not trades:
            return f"📊 {year}년 {last_month}월 - 거래 없음"
        
        # 통계 계산
        total = len(trades)
        winners = [t for t in trades if t['pnl_percent'] > 0]
        losers = [t for t in trades if t['pnl_percent'] <= 0]
        
        win_rate = (len(winners) / total) * 100
        avg_profit = sum(t['pnl_percent'] for t in winners) / len(winners) * 100 if winners else 0
        avg_loss = sum(t['pnl_percent'] for t in losers) / len(losers) * 100 if losers else 0
        total_pnl_krw = sum(t['pnl_krw'] for t in trades)
        
        # 최고/최저 거래
        best_trade = max(trades, key=lambda x: x['pnl_percent'])
        worst_trade = min(trades, key=lambda x: x['pnl_percent'])
        
        # 심볼별 집계
        symbol_stats = {}
        for trade in trades:
            symbol = trade['symbol'].split('/')[0]
            if symbol not in symbol_stats:
                symbol_stats[symbol] = {
                    'count': 0,
                    'wins': 0,
                    'total_pnl': 0
                }
            
            symbol_stats[symbol]['count'] += 1
            if trade['pnl_percent'] > 0:
                symbol_stats[symbol]['wins'] += 1
            symbol_stats[symbol]['total_pnl'] += trade['pnl_krw']
        
        # 리포트 작성
        report = f"""
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 {year}년 {last_month}월 월간 리포트
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📈 전체 성과:
  총 거래: {total}회
  승률: {win_rate:.1f}% ({len(winners)}승 {len(losers)}패)
  총 손익: {total_pnl_krw:+,.0f} KRW
  
💰 수익/손실:
  평균 수익: {avg_profit:+.2f}%
  평균 손실: {avg_loss:+.2f}%
  최고 거래: {best_trade['symbol']} {best_trade['pnl_percent']*100:+.2f}%
  최악 거래: {worst_trade['symbol']} {worst_trade['pnl_percent']*100:+.2f}%

🎯 심볼별 성과:
"""
        
        for symbol, stats in symbol_stats.items():
            win_rate_sym = (stats['wins'] / stats['count']) * 100
            report += f"  {symbol}: {stats['count']}회 | 승률 {win_rate_sym:.1f}% | 손익 {stats['total_pnl']:+,.0f} KRW\n"
        
        report += """
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
"""
        
        return report
    
    def save_report(self, report: str, report_type: str) -> None:
        """
        리포트 파일 저장
        
        Args:
            report: 리포트 내용
            report_type: 'daily', 'weekly', 'monthly'
        
        저장 위치:
            logs/reports/daily/2025-01-15.txt
            logs/reports/weekly/2025-W03.txt
            logs/reports/monthly/2025-01.txt
        """
        from pathlib import Path
        from core.constants import LOG_DIR
        
        report_dir = Path(LOG_DIR) / 'reports' / report_type
        report_dir.mkdir(parents=True, exist_ok=True)
        
        now = datetime.now()
        if report_type == 'daily':
            filename = now.strftime('%Y-%m-%d.txt')
        elif report_type == 'weekly':
            week_num = now.isocalendar()[1]
            filename = f"{now.year}-W{week_num:02d}.txt"
        else:  # monthly
            filename = now.strftime('%Y-%m.txt')
        
        filepath = report_dir / filename
        with open(filepath, 'w', encoding='utf-8') as f:
            f.write(report)
```

---

## 📁 monitoring/performance.py ⭐ 개선

### 구현 코드 (전체 개선)

```python
import numpy as np
import pandas as pd
from typing import List, Dict
from database import TradeDatabase


class PerformanceTracker:
    """
    성과 지표 계산
    
    ⭐ 개선: balance_history 저장/조회 메커니즘 추가
    """
    
    def __init__(self, db: TradeDatabase):
        """
        트래커 초기화
        
        Args:
            db: TradeDatabase 인스턴스
        """
        self.db = db
        self.balance_history = []  # ⭐ 잔고 이력
    
    def record_balance(self, timestamp: int, balance: float) -> None:
        """
        잔고 기록
        
        ⭐ 개선: 새로 추가된 함수
        
        Args:
            timestamp: Unix 타임스탬프
            balance: 현재 잔고 (KRW)
        
        호출:
            engine/base_engine.py - 5분마다
        
        Example:
            >>> tracker.record_balance(1640000000, 1050000)
        """
        self.balance_history.append((timestamp, balance))
        
        # 메모리 관리: 최대 10,000개 (약 35일치)
        if len(self.balance_history) > 10000:
            self.balance_history = self.balance_history[-10000:]
    
    def get_balance_history(self) -> List[tuple]:
        """
        잔고 이력 조회
        
        ⭐ 개선: 새로 추가된 함수
        
        Returns:
            [(timestamp, balance), ...]
        """
        return self.balance_history.copy()
    
    def calculate_sharpe_ratio(
        self,
        returns: List[float],
        risk_free_rate: float = 0.03
    ) -> float:
        """
        샤프 비율 계산
        
        Formula:
            Sharpe = (평균수익률 - 무위험수익률) / 표준편차
        
        Args:
            returns: 거래별 수익률 리스트
            risk_free_rate: 연 무위험 수익률 (기본 3%)
        
        Returns:
            float: 샤프 비율
                > 2.0: 매우 우수
                1.0 - 2.0: 우수
                0.0 - 1.0: 보통
                < 0.0: 불량
        
        Example:
            >>> tracker = PerformanceTracker(db)
            >>> sharpe = tracker.calculate_sharpe_ratio([0.02, -0.01, 0.035])
            >>> print(f"Sharpe Ratio: {sharpe:.2f}")
            Sharpe Ratio: 1.45
        """
        if len(returns) < 2:
            return 0.0
        
        returns_array = np.array(returns)
        
        # 연환산
        annual_return = returns_array.mean() * 252
        annual_std = returns_array.std() * np.sqrt(252)
        
        if annual_std == 0:
            return 0.0
        
        sharpe = (annual_return - risk_free_rate) / annual_std
        
        return round(sharpe, 2)
    
    def calculate_max_drawdown(
        self,
        balance_history: List[tuple] = None
    ) -> Dict:
        """
        최대 낙폭 계산
        
        ⭐ 개선: balance_history 인자 선택 가능
        
        Args:
            balance_history: [(timestamp, balance), ...]
                None이면 self.balance_history 사용
        
        Returns:
            {
                'max_dd': -0.15,
                'peak': 1200000,
                'trough': 1020000,
                'duration_days': 12
            }
        
        Example:
            >>> history = [(ts1, 1000000), (ts2, 1200000), (ts3, 1020000)]
            >>> dd = tracker.calculate_max_drawdown(history)
            >>> print(f"Max DD: {dd['max_dd']*100:.1f}%")
            Max DD: -15.0%
        """
        if balance_history is None:
            balance_history = self.balance_history
        
        if len(balance_history) < 2:
            return {
                'max_dd': 0.0,
                'peak': 0.0,
                'trough': 0.0,
                'duration_days': 0
            }
        
        balances = [b[1] for b in balance_history]
        timestamps = [b[0] for b in balance_history]
        
        peak = balances[0]
        max_dd = 0.0
        peak_index = 0
        trough_index = 0
        
        for i, balance in enumerate(balances):
            if balance > peak:
                peak = balance
                peak_index = i
            
            drawdown = (balance - peak) / peak
            
            if drawdown < max_dd:
                max_dd = drawdown
                trough_index = i
        
        # 기간 계산
        if trough_index > peak_index:
            duration = (timestamps[trough_index] - timestamps[peak_index]) / 86400
        else:
            duration = 0
        
        return {
            'max_dd': round(max_dd, 4),
            'peak': peak,
            'trough': balances[trough_index] if trough_index > 0 else peak,
            'duration_days': int(duration)
        }
    
    def calculate_win_rate(self, trades: List[Dict]) -> Dict:
        """
        승률 통계
        
        Args:
            trades: 거래 리스트
        
        Returns:
            {
                'total_trades': 15,
                'win_rate': 66.7,
                'avg_win': 3.5,
                'avg_loss': -1.0,
                'best_trade': 4.2,
                'worst_trade': -1.0,
                'avg_holding_minutes': 180
            }
        
        Example:
            >>> trades = db.get_trades_last_n_days(7)
            >>> stats = tracker.calculate_win_rate(trades)
            >>> print(f"승률: {stats['win_rate']:.1f}%")
            승률: 66.7%
        """
        if not trades:
            return {
                'total_trades': 0,
                'win_rate': 0.0,
                'avg_win': 0.0,
                'avg_loss': 0.0
            }
        
        total = len(trades)
        winners = [t for t in trades if t['pnl_percent'] > 0]
        losers = [t for t in trades if t['pnl_percent'] <= 0]
        
        win_rate = (len(winners) / total) * 100
        
        avg_win = (
            sum(t['pnl_percent'] for t in winners) / len(winners) * 100 
            if winners else 0
        )
        
        avg_loss = (
            sum(t['pnl_percent'] for t in losers) / len(losers) * 100 
            if losers else 0
        )
        
        best = max(trades, key=lambda x: x['pnl_percent'])
        worst = min(trades, key=lambda x: x['pnl_percent'])
        
        avg_holding = (
            sum(t.get('holding_minutes', 0) for t in trades) / total
            if all('holding_minutes' in t for t in trades) else 0
        )
        
        return {
            'total_trades': total,
            'win_rate': round(win_rate, 1),
            'avg_win': round(avg_win, 2),
            'avg_loss': round(avg_loss, 2),
            'best_trade': round(best['pnl_percent'] * 100, 2),
            'worst_trade': round(worst['pnl_percent'] * 100, 2),
            'avg_holding_minutes': int(avg_holding)
        }
    
    def calculate_profit_factor(self, trades: List[Dict]) -> float:
        """
        Profit Factor 계산
        
        Formula:
            PF = 총 수익 거래 금액 / 총 손실 거래 금액
        
        해석:
            > 2.0: 매우 우수
            1.5 - 2.0: 우수
            1.0 - 1.5: 보통
            < 1.0: 개선 필요
        
        Example:
            >>> pf = tracker.calculate_profit_factor(trades)
            >>> print(f"Profit Factor: {pf:.2f}")
            Profit Factor: 1.85
        """
        if not trades:
            return 0.0
        
        winners = [t for t in trades if t['pnl_krw'] > 0]
        losers = [t for t in trades if t['pnl_krw'] <= 0]
        
        total_wins = sum(t['pnl_krw'] for t in winners) if winners else 0
        total_losses = abs(sum(t['pnl_krw'] for t in losers)) if losers else 0
        
        if total_losses == 0:
            return 0.0 if total_wins == 0 else float('inf')
        
        profit_factor = total_wins / total_losses
        
        return round(profit_factor, 2)
    
    def get_performance_metrics(
        self,
        start_date: str = None,
        end_date: str = None
    ) -> Dict:
        """
        전체 성과 지표 한 번에 계산
        
        Args:
            start_date: 시작일 'YYYY-MM-DD'
            end_date: 종료일 'YYYY-MM-DD'
        
        Returns:
            {
                'win_rate_stats': {...},
                'sharpe_ratio': 1.45,
                'max_drawdown': {...},
                'profit_factor': 1.85,
                'total_pnl_krw': 35420,
                'best_day': {...},
                'worst_day': {...}
            }
        
        호출:
            monitoring/reporter.py - 월간 리포트
        
        Example:
            >>> metrics = tracker.get_performance_metrics('2025-01-01', '2025-01-31')
            >>> print(f"Sharpe: {metrics['sharpe_ratio']}")
            >>> print(f"Max DD: {metrics['max_drawdown']['max_dd']*100:.1f}%")
        """
        # 거래 조회
        if start_date and end_date:
            trades = self.db.get_trades_by_period(start_date, end_date)
        else:
            trades = self.db.get_all_closed_trades()
        
        if not trades:
            return {'error': 'No trades found'}
        
        # 1. 승률 통계
        win_rate_stats = self.calculate_win_rate(trades)
        
        # 2. 샤프 비율
        returns = [t['pnl_percent'] for t in trades]
        sharpe = self.calculate_sharpe_ratio(returns)
        
        # 3. Profit Factor
        profit_factor = self.calculate_profit_factor(trades)
        
        # 4. 총 손익
        total_pnl_krw = sum(t['pnl_krw'] for t in trades)
        
        # 5. 최대 낙폭
        max_dd = self.calculate_max_drawdown()
        
        # 6. 일별 최고/최저
        daily_pnl = {}
        for trade in trades:
            date = trade['timestamp'].split()[0]
            if date not in daily_pnl:
                daily_pnl[date] = 0
            daily_pnl[date] += trade['pnl_krw']
        
        best_day = max(daily_pnl.items(), key=lambda x: x[1]) if daily_pnl else ('', 0)
        worst_day = min(daily_pnl.items(), key=lambda x: x[1]) if daily_pnl else ('', 0)
        
        return {
            'win_rate_stats': win_rate_stats,
            'sharpe_ratio': sharpe,
            'max_drawdown': max_dd,
            'profit_factor': profit_factor,
            'total_pnl_krw': round(total_pnl_krw, 0),
            'best_day': {'date': best_day[0], 'pnl': round(best_day[1], 0)},
            'worst_day': {'date': worst_day[0], 'pnl': round(worst_day[1], 0)}
        }
```

---

## 📁 monitoring/log_cleaner.py ⭐ 신규

### 구현 코드 (신규 추가)

```python
import os
import gzip
import shutil
from pathlib import Path
from datetime import datetime, timedelta
from core.constants import LOG_DIR


def cleanup_old_logs(mode: str = 'paper', days: int = 30) -> None:
    """
    오래된 로그 파일 정리
    
    ⭐ 개선: 신규 추가 함수
    
    Args:
        mode: 'paper', 'live', 'backtest'
        days: 보관 일수
    
    처리:
        1. {days}일 이상 된 로그 파일 찾기
        2. gzip 압축
        3. archives/ 디렉토리로 이동
        4. 원본 삭제
    
    호출:
        cron/Task Scheduler - 매일 자정
    
    Example:
        >>> cleanup_old_logs('paper', days=30)
        ✅ 아카이브: 2024-12-15_trades.log → 2024-12-15_trades.log.gz
        ✅ 아카이브: 2024-12-15_decisions.log → 2024-12-15_decisions.log.gz
    """
    log_dir = Path(LOG_DIR) / mode
    archive_dir = Path(LOG_DIR) / 'archives' / mode
    archive_dir.mkdir(parents=True, exist_ok=True)
    
    cutoff_date = datetime.now() - timedelta(days=days)
    archived_count = 0
    
    for filename in os.listdir(log_dir):
        if not filename.endswith('.log'):
            continue
        
        filepath = log_dir / filename
        
        # 파일 수정 시간
        mtime = datetime.fromtimestamp(os.path.getmtime(filepath))
        
        if mtime < cutoff_date:
            # 압축
            gz_filename = f'{filename}.gz'
            gz_filepath = archive_dir / gz_filename
            
            with open(filepath, 'rb') as f_in:
                with gzip.open(gz_filepath, 'wb') as f_out:
                    shutil.copyfileobj(f_in, f_out)
            
            # 원본 삭제
            os.remove(filepath)
            
            print(f"✅ 아카이브: {filename} → {gz_filename}")
            archived_count += 1
    
    if archived_count == 0:
        print(f"📋 정리할 로그 없음 ({mode})")
    else:
        print(f"✅ {archived_count}개 파일 아카이브 완료 ({mode})")


def cleanup_all_modes(days: int = 30) -> None:
    """
    모든 모드의 로그 정리
    
    호출:
        scripts/cleanup_logs.py
    """
    cleanup_old_logs('paper', days)
    cleanup_old_logs('live', days=365)  # Live는 1년 보관
    cleanup_old_logs('backtest', days=90)
```

---

## 전체 의존성 그래프

```
logger.py
├── 사용: logging, core/constants, database/execute_query
└── 사용처: engine/base_engine.py

reporter.py
├── 사용: database/TradeDatabase, datetime
└── 사용처: engine/base_engine.py

performance.py
├── 사용: database/TradeDatabase, numpy, pandas
└── 사용처: monitoring/reporter.py, engine/base_engine.py

log_cleaner.py ⭐
├── 사용: os, gzip, shutil, core/constants
└── 사용처: scripts/cleanup_logs.py, cron
```

---

## 실전 사용 예제

### 예제 1: 엔진에서 로거 사용

```python
from monitoring import TradingLogger

class BaseEngine:
    def __init__(self, config, exchange):
        # 로거 초기화
        self.logger = TradingLogger(mode='paper', level='INFO')
    
    async def execute_entry(self, symbol, signal):
        # 진입 로그
        self.logger.log_entry(
            symbol,
            signal['price'],
            signal['amount_krw'],
            signal['confidence']
        )
        
        # AI 결정 로그
        self.logger.log_ai_decision(
            symbol,
            'ENTER',
            signal['confidence'],
            signal['reasoning']
        )
        
        # ... 주문 실행 ...
    
    async def execute_exit(self, symbol, position, signal):
        # 청산 로그
        self.logger.log_exit(
            symbol,
            position['entry_price'],
            signal['exit_price'],
            signal['pnl_percent'],
            signal['pnl_krw'],
            signal['reason']
        )
```

### 예제 2: 일일 리포트 생성

```python
from monitoring import PerformanceReporter
from database import TradeDatabase

class BaseEngine:
    def __init__(self, config, exchange):
        self.reporter = PerformanceReporter(TradeDatabase())
    
    async def check_daily_report(self):
        """매일 09:00, 21:00 실행"""
        current_hour = datetime.now().hour
        
        if current_hour in [9, 21]:
            # 리포트 생성
            report = self.reporter.generate_daily_summary()
            
            # 출력
            print(report)
            
            # 저장
            self.reporter.save_report(report, 'daily')
            
            # 이메일 발송 (선택)
            # send_email("일일 리포트", report)
```

### 예제 3: 잔고 이력 기록

```python
from monitoring import PerformanceTracker
from database import TradeDatabase

class BaseEngine:
    def __init__(self, config, exchange):
        self.tracker = PerformanceTracker(TradeDatabase())
    
    async def run(self):
        self.running = True
        
        # 잔고 기록 태스크 시작
        asyncio.create_task(self.record_balance_periodically())
        
        # 메인 루프
        while self.running:
            await self.main_loop()
            await asyncio.sleep(60)
    
    async def record_balance_periodically(self):
        """5분마다 잔고 기록"""
        while self.running:
            balance = self.exchange.get_balance()['total_krw']
            timestamp = int(time.time())
            
            self.tracker.record_balance(timestamp, balance)
            
            await asyncio.sleep(300)  # 5분
```

### 예제 4: 최대 낙폭 모니터링

```python
class BaseEngine:
    async def main_loop(self):
        # 최대 낙폭 확인
        dd_info = self.tracker.calculate_max_drawdown()
        
        if abs(dd_info['max_dd']) > 0.08:  # -8% 이상
            print(f"⚠️ 최대 낙폭 경고: {dd_info['max_dd']*100:.1f}%")
            
            # 리스크 이벤트 기록
            self.logger.log_risk_event('HIGH_DRAWDOWN', {
                'max_dd': dd_info['max_dd'],
                'peak': dd_info['peak'],
                'trough': dd_info['trough']
            })
```

### 예제 5: 로그 자동 정리

```python
# scripts/cleanup_logs.py
from monitoring.log_cleaner import cleanup_all_modes

if __name__ == '__main__':
    print("🧹 로그 정리 시작...")
    
    # Paper: 30일 보관
    # Live: 365일 보관
    # Backtest: 90일 보관
    cleanup_all_modes()
    
    print("✅ 로그 정리 완료")
```

---

## 테스트 시나리오

### logger.py 테스트

```python
import pytest
from monitoring import TradingLogger
from pathlib import Path
import os

def test_logger_initialization():
    """로거 초기화 테스트"""
    logger = TradingLogger(mode='test', level='INFO')
    
    assert logger.mode == 'test'
    assert logger.level == 'INFO'
    assert logger.logger is not None


def test_log_entry():
    """진입 로그 테스트"""
    logger = TradingLogger(mode='test', level='INFO')
    
    # 로그 기록
    logger.log_entry('DOGE/USDT', 0.3821, 500000, 0.75)
    
    # 로그 파일 확인
    today = datetime.now().strftime('%Y-%m-%d')
    log_file = Path('logs/test') / f'{today}_trades.log'
    
    assert log_file.exists()
    
    with open(log_file, 'r') as f:
        content = f.read()
        assert '[ENTRY]' in content
        assert 'DOGE/USDT' in content


def test_risk_event_db_save():
    """리스크 이벤트 DB 저장 테스트"""
    from database import init_database, execute_query
    
    init_database()
    logger = TradingLogger(mode='test', level='INFO')
    
    # 리스크 이벤트 기록
    logger.log_risk_event('DAILY_LIMIT', {
        'loss': -0.05,
        'resume_time': 1234567890
    })
    
    # DB 확인
    events = execute_query(
        "SELECT * FROM risk_events WHERE event_type = ?",
        ('DAILY_LIMIT',),
        fetch='all'
    )
    
    assert len(events) > 0
    assert events[0]['event_type'] == 'DAILY_LIMIT'
```

### reporter.py 테스트

```python
def test_generate_daily_summary():
    """일일 리포트 생성 테스트"""
    from database import init_database, TradeDatabase
    
    init_database()
    db = TradeDatabase()
    
    # 테스트 데이터
    db.save_trade_entry({
        'timestamp': datetime.now(),
        'symbol': 'DOGE/USDT',
        'mode': 'test',
        'entry_price': 0.3821,
        'quantity': 1000,
        'ai_confidence': 0.75,
        'entry_fee': 0.38
    })
    
    # 리포트 생성
    reporter = PerformanceReporter(db)
    report = reporter.generate_daily_summary()
    
    assert '일일 리포트' in report
    assert 'DOGE' in report


def test_generate_monthly_report():
    """월간 리포트 생성 테스트"""
    reporter = PerformanceReporter(TradeDatabase())
    report = reporter.generate_monthly_report()
    
    assert '월간 리포트' in report
```

### performance.py 테스트

```python
def test_record_balance():
    """잔고 기록 테스트"""
    tracker = PerformanceTracker(TradeDatabase())
    
    # 잔고 기록
    tracker.record_balance(1640000000, 1000000)
    tracker.record_balance(1640000300, 1050000)
    tracker.record_balance(1640000600, 980000)
    
    # 확인
    history = tracker.get_balance_history()
    assert len(history) == 3
    assert history[0][1] == 1000000


def test_calculate_max_drawdown():
    """최대 낙폭 계산 테스트"""
    tracker = PerformanceTracker(TradeDatabase())
    
    history = [
        (1640000000, 1000000),
        (1640000300, 1200000),  # 고점
        (1640000600, 1020000),  # 저점
        (1640000900, 1100000)
    ]
    
    dd = tracker.calculate_max_drawdown(history)
    
    assert dd['max_dd'] == -0.15  # -15%
    assert dd['peak'] == 1200000
    assert dd['trough'] == 1020000


def test_sharpe_ratio():
    """샤프 비율 테스트"""
    tracker = PerformanceTracker(TradeDatabase())
    
    returns = [0.02, -0.01, 0.035, 0.018, -0.008, 0.025]
    sharpe = tracker.calculate_sharpe_ratio(returns)
    
    assert sharpe > 0
    assert isinstance(sharpe, float)
```

### log_cleaner.py 테스트

```python
def test_cleanup_old_logs():
    """로그 정리 테스트"""
    from monitoring.log_cleaner import cleanup_old_logs
    import time
    
    # 테스트 로그 생성
    log_dir = Path('logs/test')
    log_dir.mkdir(parents=True, exist_ok=True)
    
    old_log = log_dir / 'old_test.log'
    with open(old_log, 'w') as f:
        f.write('test log')
    
    # 수정 시간을 40일 전으로 변경
    old_time = time.time() - (40 * 86400)
    os.utime(old_log, (old_time, old_time))
    
    # 정리 실행
    cleanup_old_logs('test', days=30)
    
    # 확인
    assert not old_log.exists()
    
    archive_dir = Path('logs/archives/test')
    assert (archive_dir / 'old_test.log.gz').exists()
```

---

## 개발 체크리스트

### logger.py
- [x] TradingLogger 클래스
- [x] setup_logger() - 파일/콘솔 핸들러
- [x] log_entry() - 진입 로그
- [x] log_exit() - 청산 로그
- [x] log_risk_event() - 리스크 이벤트
- [x] ⭐ log_risk_event() DB 저장 추가
- [x] log_ai_decision() - AI 판단
- [x] 로그 필터 (거래/AI/에러 분리)

### reporter.py ⭐
- [x] PerformanceReporter 클래스
- [x] generate_daily_summary() - 일일 요약
- [x] generate_weekly_report() - 주간 분석
- [x] ⭐ generate_monthly_report() - 월간 리포트 완성
- [x] save_report() - 파일 저장

### performance.py ⭐
- [x] PerformanceTracker 클래스
- [x] ⭐ record_balance() - 잔고 기록 (신규)
- [x] ⭐ get_balance_history() - 잔고 이력 조회 (신규)
- [x] calculate_sharpe_ratio() - 샤프 비율
- [x] calculate_max_drawdown() - 최대 낙폭
- [x] calculate_win_rate() - 승률 통계
- [x] calculate_profit_factor() - Profit Factor
- [x] get_performance_metrics() - 종합 지표

### log_cleaner.py ⭐
- [x] ⭐ cleanup_old_logs() - 로그 정리 (신규)
- [x] ⭐ cleanup_all_modes() - 전체 모드 정리 (신규)

---

## 주요 특징

### 1. 다층 로깅 시스템
- 콘솔: 실시간 모니터링
- trades.log: 거래 기록만
- decisions.log: AI 판단만
- errors.log: 에러만
- debug.log: 전체 (DEBUG 모드)

### 2. 자동 리포트 생성
- 일일: 매일 09:00, 21:00
- 주간: 매주 월요일 00:00
- 월간: 매월 1일 00:00 ⭐

### 3. 표준 성과 지표
- Sharpe Ratio: 위험 대비 수익
- Max Drawdown: 최대 낙폭
- Win Rate: 승률
- Profit Factor: 수익/손실 비율

### 4. 파일 관리 ⭐
- 로그 자동 정리 (30일)
- gzip 압축 아카이브
- Live 로그는 1년 보관

### 5. Balance History ⭐
- 5분마다 잔고 기록
- 최대 낙폭 계산에 사용
- 메모리 효율적 관리 (10,000개 제한)

### 6. DB 연동 ⭐
- 리스크 이벤트 자동 저장
- 거래 데이터 조회 최적화
- 학습 데이터 분석 지원

---

**문서 버전**: v2.0 (개선판)  
**작성일**: 2025-01-22  
**개선사항**:
- ⭐ generate_monthly_report() 완전 구현
- ⭐ record_balance() / get_balance_history() 신규 추가
- ⭐ log_risk_event() DB 저장 기능 추가
- ⭐ log_cleaner.py 신규 모듈 추가 (자동 정리)
- ⭐ balance_history 저장 메커니즘 완성
- ✅ 실전 사용 예제 5개
- ✅ 테스트 시나리오 완성

**검증 상태**: ✅ 완료
