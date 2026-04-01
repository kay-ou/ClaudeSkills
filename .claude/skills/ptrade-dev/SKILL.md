---
name: ptrade-dev
version: 1.0.0
description: |
  PTrade/SimTradeLab strategy development guardrails. Auto-loads PTrade API reference,
  enforces platform constraints (no f-string, no import io/sys), validates lifecycle
  function usage, and prevents common API misuse. Use when writing, editing, or reviewing
  any PTrade strategy code or SimTradeLab backtest code.
  TRIGGER: when editing files in strategies/, src/simtradelab/ptrade/, or any .py file
  that imports from simtradelab or uses PTrade API functions (initialize, handle_data,
  set_universe, get_history, order, etc).
allowed-tools:
  - Read
  - Edit
  - Write
  - Bash
  - Grep
  - Glob
---

# PTrade Strategy Development Skill

You are writing code for the PTrade quantitative trading platform or its local simulator SimTradeLab.
This skill ensures you NEVER make API errors by providing the complete reference inline.

## CRITICAL PLATFORM CONSTRAINTS

These constraints apply to ALL code that runs on PTrade (NOT local-only files like research/run_local_backtest.py):

1. **NO f-strings** — Use `%` formatting or `.format()`. PTrade's Python does not support f-strings.
   ```python
   # WRONG
   log.info(f"price: {price}")
   # CORRECT
   log.info("price: %s" % price)
   log.info("price: {}".format(price))
   ```

2. **NO `import io` or `import sys`** — These modules are blocked on PTrade.

3. **NO walrus operator `:=`** — Not supported.

4. **NO `match/case`** — Not supported.

5. **Code format**: PTrade uses `.SS` (Shanghai) and `.SZ` (Shenzhen) suffixes, but Order objects return `.XSHG` / `.XSHE` suffixes. Be aware of this difference.

6. **Global object `g`**: Use `g.xxx` for cross-function state. Variables starting with `__` are private and won't be persisted.

7. **`log` object**: Use `log.info()`, `log.warning()`, `log.error()` etc. Never `print()`.

## STRATEGY LIFECYCLE (MANDATORY KNOWLEDGE)

A strategy has exactly 2 required functions and 5 optional ones:

```
initialize(context)          # REQUIRED — runs once at startup
handle_data(context, data)   # REQUIRED — runs every bar (daily/minute)
before_trading_start(context, data)  # optional — runs before market open
after_trading_end(context, data)     # optional — runs after market close
tick_data(context, data)     # optional — runs every 3s (live trading only)
on_order_response(context, order_list)  # optional — order callback (live only)
on_trade_response(context, trade_list)  # optional — trade callback (live only)
```

### Lifecycle Timing
- **initialize**: runs ONCE at strategy start
- **before_trading_start**: backtest 8:30, live 9:10 (configurable)
- **handle_data**: backtest 9:31-15:00 (minute) or 15:00 (daily); live 9:30-14:59
- **after_trading_end**: ~15:30
- **tick_data**: 9:30-14:59, every 3s (live only)

## API FUNCTION LIFECYCLE RESTRICTIONS

**CRITICAL**: Each API can ONLY be called from specific lifecycle functions. Calling from the wrong function will error.

### initialize-ONLY APIs (setup functions)
```
set_universe(securities)          # Set stock pool
set_benchmark(benchmark)          # Set benchmark index
set_commission(commission)        # Set commission (backtest only)
set_fixed_slippage(slippage)      # Set fixed slippage (backtest only)
set_slippage(slippage)            # Set slippage (backtest only)
set_volume_ratio(ratio)           # Set volume ratio (backtest only)
set_limit_mode(mode)              # Set limit mode (backtest only)
set_yesterday_position(positions) # Set initial positions (backtest only)
set_parameters(params)            # Set strategy params
run_daily(func, time)             # Schedule daily function
run_interval(func, interval)      # Schedule interval function (live only)
set_future_commission(commission)  # Futures commission (backtest only)
set_margin_rate(security, rate)   # Futures margin (backtest/live)
permission_test(account, end_date)# Permission check (live only)
create_dir(user_path)             # Create directory (live only)
```

### handle_data / tick_data APIs (trading functions)
```
order(security, amount, limit_price=None)           # Buy/sell by amount
order_target(security, target_amount, limit_price=None)  # Target amount
order_value(security, value, limit_price=None)      # Buy/sell by value
order_target_value(security, target_value, limit_price=None)  # Target value
order_market(security, amount)                      # Market order (live only)
cancel_order(order_id)                              # Cancel order
cancel_order_ex(order_id)                           # Cancel order extended (live)
order_tick(security, amount, limit_price, tick_type) # Tick order (live only)
```

### after_trading_end APIs
```
after_trading_order(security, amount, limit_price)  # After-hours order (live)
after_trading_cancel_order(order_id)                # Cancel after-hours (live)
get_trades_file()                                   # Get trade file (backtest)
get_deliver(start_date, end_date)                   # Delivery records (live)
get_fundjour(start_date, end_date)                  # Fund journal (live)
send_email(...)                                     # Send email (live)
send_qywx(...)                                      # Send WeChat (live)
```

### Universal APIs (callable from ANY lifecycle function)
```
# Market data
get_history(count, frequency, field, security_list, fq, include, fill, is_dict, start_date, end_date)
get_price(security, start_date, end_date, frequency, fields, count)
get_snapshot(security_list)         # Live only, handle_data/tick_data
get_gear_price(security_list)      # Live only, handle_data/tick_data

# Trading info
get_position(security)             # Get position for one stock
get_positions(security_list)       # Get positions for multiple stocks
get_open_orders(security=None)     # Get pending orders
get_order(order_id)                # Get specific order
get_orders(security=None)          # Get all orders today
get_trades(security=None)          # Get trades today

# Stock info
get_stock_name(security_list)
get_stock_info(security_list)
get_stock_status(security_list)
get_stock_exrights(security_list)
get_stock_blocks(security_list)
get_index_stocks(index_code)
get_industry_stocks(industry_code)
get_fundamentals(stocks, table, fields, date)
get_Ashares(date)
check_limit(security, query_date=None)

# Date/Calendar
get_trading_day(day=0)
get_all_trades_days(date=None)
get_trade_days(start_date, end_date)

# Technical indicators
get_MACD(close, short=12, long=26, m=9)
get_KDJ(high, low, close, n=9, m1=3, m2=3)
get_RSI(close, n=6)
get_CCI(high, low, close, n=14)

# Utility
log.info/warning/error/debug/critical
is_trade()                         # True if live trading
get_user_name()
get_research_path()
```

## KEY OBJECTS REFERENCE

### context.portfolio (Portfolio)
```python
context.portfolio.cash              # Available cash (excludes frozen)
context.portfolio.positions         # dict: {code: Position}
context.portfolio.portfolio_value   # Total value (cash + positions)
context.portfolio.positions_value   # Positions value only
context.portfolio.capital_used      # Used capital
context.portfolio.returns           # Return ratio vs initial capital
context.portfolio.pnl               # Total P&L
context.portfolio.start_date        # Start date
```

### Position (stock)
```python
pos = context.portfolio.positions[code]
pos.sid                # Stock code
pos.amount             # Total shares held
pos.enable_amount      # Sellable shares (T+1)
pos.last_sale_price    # Latest price
pos.cost_basis         # Average cost
pos.today_amount       # Bought today (backtest only)
pos.business_type      # Position type
```

### data (in handle_data)
```python
# data is a dict: {stock_code: SecurityUnitData}
bar = data[security]
bar['open']    # or bar.open
bar['close']   # or bar.close
bar['high']    # or bar.high
bar['low']     # or bar.low
bar['volume']  # or bar.volume
bar['money']   # or bar.money
bar['price']   # = close
bar['dt']      # datetime
```

### Order object
```python
order_obj.id       # Order ID
order_obj.dt       # Order time (datetime.datetime)
order_obj.limit    # Limit price
order_obj.symbol   # Code (note: uses .XSHG/.XSHE suffix!)
order_obj.amount   # Amount: positive=buy, negative=sell
```

### context.blotter
```python
context.blotter.current_dt  # Current bar datetime (Beijing time)
```

## get_history DETAILED REFERENCE

This is the MOST commonly misused API. Pay close attention:

```python
get_history(
    count,                    # Number of bars to fetch
    frequency='1d',           # '1d' or '1m'
    field='close',            # Single field string or list
    security_list=None,       # Stock code(s) - string or list
    fq=None,                  # 'pre'=forward adj, 'post'=backward adj, None=raw
    include=False,            # True=include current bar
    fill='nan',               # Fill method: 'nan', 'pre' (forward fill)
    is_dict=False,            # True=return dict of DataFrames
    start_date=None,          # Alternative to count
    end_date=None             # End date for range query
)
```

**Return type varies:**
- Single field + single stock: `pd.DataFrame` with column = field name
- Single field + multiple stocks: `pd.DataFrame` with columns = stock codes
- Multiple fields + `is_dict=False`: `PanelLike` object (dict-like, key=field)
- Multiple fields + `is_dict=True`: dict of DataFrames

**Common patterns:**
```python
# Get 20-day close for one stock
df = get_history(20, '1d', 'close', '600570.SS', fq='pre', include=False)
ma20 = df['close'].mean()

# Get OHLCV for one stock
data = get_history(20, '1d', ['open','high','low','close','volume'], '600570.SS', fq='pre')
closes = data['close']['600570.SS']  # PanelLike access

# Get close for multiple stocks
df = get_history(20, '1d', 'close', ['600570.SS', '000001.SZ'])
# df.columns = ['600570.SS', '000001.SZ']
```

## get_price DETAILED REFERENCE

```python
get_price(
    security,                 # Stock code (string)
    start_date=None,          # 'YYYY-MM-DD' or 'YYYY-MM-DD HH:MM:SS'
    end_date=None,            # 'YYYY-MM-DD' or 'YYYY-MM-DD HH:MM:SS'
    frequency='1d',           # '1d' or '1m'
    fields=None,              # List of fields, default all OHLCV
    count=None                # Alternative to date range
)
```

**Returns:** `pd.DataFrame` with DatetimeIndex, columns = requested fields

## COMMON MISTAKES TO PREVENT

1. **Calling `order()` in `initialize()`** — Trading functions only work in `handle_data`/`tick_data`
2. **Using `set_commission()` in `handle_data()`** — Setup functions only work in `initialize()`
3. **Assuming `data[security]` always exists** — Only stocks in `set_universe()` are in `data`
4. **Forgetting T+1 rule** — `pos.enable_amount` may be 0 for stocks bought today
5. **Using `.XSHG`/`.XSHE` codes with `order()`** — Use `.SS`/`.SZ` format for orders
6. **Not checking `pos.enable_amount` before selling** — Will fail if no sellable shares
7. **Calling `get_snapshot()` in backtest** — Only available in live trading
8. **Using f-strings** — PTrade does NOT support f-strings!
9. **`get_history` with `include=True`** — Current bar data may be incomplete
10. **Negative amount in `order()`** — Negative = sell, positive = buy

## SIMTRADELAB LOCAL-ONLY DIFFERENCES

When writing code for SimTradeLab local backtest (research/run_local_backtest.py):
- f-strings ARE allowed (local Python 3.9+)
- `import io/sys` ARE allowed
- Data comes from local parquet/CSV files via DataContext
- API behavior is simulated to match PTrade as closely as possible

## SELF-CHECK BEFORE SUBMITTING CODE

Before finishing any PTrade strategy code, verify:
- [ ] No f-strings in PTrade-targeted code
- [ ] No `import io` or `import sys` in PTrade-targeted code
- [ ] All API calls are in the correct lifecycle function
- [ ] Stock codes use correct suffix (.SS/.SZ)
- [ ] `set_universe()` is called in `initialize()`
- [ ] Trading functions (`order` etc.) are only in `handle_data`/`tick_data`
- [ ] Check `enable_amount` before selling
- [ ] Using `log.info()` not `print()`
- [ ] `get_history` parameters are correct (especially `fq` and `include`)
