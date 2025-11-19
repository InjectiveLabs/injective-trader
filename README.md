# Injective Trader SDK

Injective Trader is a high-performance, enterprise-grade framework for developing and deploying algorithmic trading strategies on Injective Protocol. It provides a comprehensive infrastructure that handles everything from real-time market data streaming to order execution, allowing developers to focus purely on trading logic. For a complete walkthrough of architecture, setup, and advanced guides, read the official [Injective Trader documentation](https://docs.injective.network/developers-defi/injective-trader).

## Features

- **Complete Trading Framework**: Handle everything from market data streaming to order execution
- **Simplified Strategy Development**: Focus purely on trading logic without blockchain complexities
- **Built-in Reliability**: Automatic reconnection, recovery mechanisms, and comprehensive error handling
- **Multi-Account Support**: Support for both direct trading and authz delegation
- **Real-time Data**: Live market data streaming and order book management
- **Performance Optimization**: Intelligent transaction batching and fee management
- **Enterprise-Ready**: Position tracking, risk management, and performance analytics
- **Flexible Component System**: Modular design with optional components like Redis/Valkey listeners
- **Advanced Order Management**: Support for limit orders, market orders, and batch operations

## Installation

### Prerequisites

- Python 3.9 or higher
- Git (for cloning the repository)
- Required Python packages will be installed automatically

### Setup

```bash
# Install from main branch
pip install git+https://github.com/InjectiveLabs/injective-trader.git

# Or install from specific branch
pip install git+https://github.com/InjectiveLabs/injective-trader.git@branch_name
```

⚠️ **Note**: This SDK requires `injective-py` version 1.9.0. If you encounter compatibility issues, please update your injective-py installation following the [official instructions](https://github.com/InjectiveLabs/sdk-python).

## Quick Start

### 1. Environment Setup

Before running the framework, you must export your Injective private keys using the bot-specific naming convention:

```bash
# For a bot named "MyBot", export private keys as:
export MyBot_GRANTER_INJECTIVE_PRIVATE_KEY=your_granter_private_key_here
export MyBot_GRANTEE_0_INJECTIVE_PRIVATE_KEY=your_first_grantee_private_key_here  
export MyBot_GRANTEE_1_INJECTIVE_PRIVATE_KEY=your_second_grantee_private_key_here
# ... add more grantees as needed

# The pattern is: {BOT_NAME}_GRANTER_INJECTIVE_PRIVATE_KEY for granter
# And: {BOT_NAME}_GRANTEE_{INDEX}_INJECTIVE_PRIVATE_KEY for grantees
```

⚠️ **Security Warning**: Never share your private key or commit it to version control. Consider using a `.env` file with appropriate permissions for development.

### 2. Configuration

Create a configuration file `config.yaml`:

```yaml
Exchange: Helix
ConsoleLevel: INFO
FileLevel: DEBUG

Components:
  Initializer:
    BotName: "MyBot"
    Network: mainnet
    MarketTickers:
      - INJ/USDT PERP
  ChainListener:
    ReconnectionDelay: 5
    LargeGapThreshold: 2
  MessageBroadcaster:
    ErrorCodesJson: config/error_codes.json
    GranteePool:
      MaxPendingTxs: 5
      ErrorThreshold: 3
      BlockDuration: 300
      RotationInterval: 1
    RefreshInterval: 300
    Batch:
      MaxBatchSize: 15
      MinBatchSize: 3
      MaxGasLimit: 5000000
      MaxBatchDelay: 0.5

Strategies:
  MyStrategy:
    Name: "MyStrategy"
    Class: "MyStrategy"
    MarketIds:
      - "0x17ef48032cb24375ba7c2e39f384e56433bcab20cbee9a7357e4cba2eb00abe6"  # INJ/USDT PERP
    AccountAddresses:
      - "inj1youraccount..."
    TradingAccount: "inj1youraccount..."  # For direct trading
    CIDPrefix: "my_strategy"
    Parameters:
      OrderSize: 0.1
      MaxPosition: 1.0

RetryConfig:
  DefaultRetry:
    max_attempts: 3
    base_delay: 1.0
    max_delay: 32.0
    jitter: true
    timeout: 30.0
    error_threshold: 10
    error_window: 60
```

### 3. Create Your Strategy

Create your strategy file (can be placed anywhere in your project):

```python
# MyStrategy.py (can be placed anywhere in your project)
from decimal import Decimal
from typing import Dict
from injective_trader.strategy.strategy import Strategy, StrategyResult
from injective_trader.domain.account.order import Order
from injective_trader.utils.enums import UpdateType, Side, MarketType

class MyStrategy(Strategy):
    def __init__(self, logger, config):
        super().__init__(logger, config)
        
    def on_initialize(self, accounts: Dict, markets: Dict) -> StrategyResult:
        """
        Initialize strategy-specific state and parameters.
        Called once before strategy starts processing updates.
        """
        # Get strategy parameters from config
        params = self.config.get("Parameters", {})
        self.order_size = Decimal(str(params.get("OrderSize", "0.1")))
        self.max_position = Decimal(str(params.get("MaxPosition", "1.0")))
        
        # Initialize strategy state
        self.logger.info(f"Strategy {self.name} initialized with order size: {self.order_size}")
        
        # Return None if no initial orders needed
        return None
        
    async def _execute_strategy(self, update_type: UpdateType, processed_data: Dict) -> StrategyResult:
        """
        Main strategy execution logic.
        
        Args:
            update_type: Type of update (OnOrderbook, OnPosition, etc.)
            processed_data: Processed update data
            
        Returns:
            StrategyResult with orders, cancellations, etc.
        """
        result = StrategyResult()
        
        # Handle different update types
        if update_type == UpdateType.OnOrderbook:
            # React to orderbook changes
            market = processed_data["market"]
            bid, ask = market.orderbook.tob()
            
            if bid and ask:
                # Simple market making logic
                spread = ask - bid
                mid_price = (bid + ask) / Decimal("2")
                
                # Create buy order below mid
                buy_order = Order(
                    market_id=market.market_id,
                    market_type=market.market_type,
                    subaccount_id=list(self.accounts.values())[0].subaccounts.keys().__iter__().__next__(),
                    order_side=Side.BUY,
                    price=mid_price - spread / Decimal("4"),
                    quantity=self.order_size,
                )
                
                # Create sell order above mid
                sell_order = Order(
                    market_id=market.market_id,
                    market_type=market.market_type,
                    subaccount_id=list(self.accounts.values())[0].subaccounts.keys().__iter__().__next__(),
                    order_side=Side.SELL,
                    price=mid_price + spread / Decimal("4"),
                    quantity=self.order_size,
                )
                
                result.orders = [buy_order, sell_order]
                
        elif update_type == UpdateType.OnPosition:
            # React to position changes
            position = processed_data["position"]
            self.logger.info(f"Position updated: {position.quantity}")
            
        return result
```

### 4. Create Main Application

Create your main application file:

```python
# main.py
import argparse
import asyncio
import sys
from pathlib import Path
from injective_trader import TraderClient

# Import your strategy (adjust import path as needed)
try:
    from MyStrategy import MyStrategy
except ImportError:
    MyStrategy = None

async def main(name: str, config_path: str, log_path: str = None, debug: bool = False, network: str = "mainnet"):
    """Main entry point for the trading bot."""
    
    # Initialize client
    client = TraderClient(
        config_path=config_path,
        network=network,
        debug=debug,
    )
    
    # Setup logging handlers manually
    if log_path:
        from injective_trader.utils.logger import create_file_handler
        file_handler = create_file_handler(log_file=log_path, level="DEBUG")
        client.add_log_handler(file_handler)
    
    # Add optional components if needed
    # client.add_component("ValkeyListener")
    
    await client.initialize()
    
    # Add your strategy
    if MyStrategy is None:
        print("MyStrategy class not found. Ensure it is defined.")
        sys.exit(1)
        
    await client.add_strategy("MyStrategy", MyStrategy)
    
    # Start the client
    try:
        client.logger.info(f"Starting {name} trading bot...")
        await client.start()
    except KeyboardInterrupt:
        client.logger.info("Received shutdown signal, stopping the bot...")
    except Exception as e:
        client.logger.error(f"An error occurred: {e}")
        raise
    finally:
        client.logger.info("Shutting down the client...")
        await client.stop()

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Injective Trading Bot")
    parser.add_argument("name", help="Bot name")
    parser.add_argument("config_path", help="Path to config file (YAML)")
    parser.add_argument("--log_path", default=None, help="Path for log file")
    parser.add_argument("--network", default="mainnet", choices=["mainnet", "testnet"])
    parser.add_argument("--debug", default=True, action=argparse.BooleanOptionalAction)
    
    args = parser.parse_args()
    
    # Validate paths
    if not Path(args.config_path).exists():
        print(f"Error: Config file not found: {args.config_path}")
        sys.exit(1)
    
    # Create log directory if needed
    if args.log_path:
        Path(args.log_path).parent.mkdir(parents=True, exist_ok=True)
    
    try:
        asyncio.run(main(
            name=args.name,
            config_path=args.config_path,
            log_path=args.log_path,
            network=args.network,
            debug=args.debug
        ))
    except KeyboardInterrupt:
        print("Bot stopped by user")
    except Exception as e:
        print(f"Error: {e}")
        sys.exit(1)
```

### 5. Run Your Strategy

```bash
python main.py MyBot config.yaml --log_path logs/strategy.log --network mainnet
```

## Advanced Features

### Authorization Trading (Authz)

For trading on behalf of other accounts using Injective's authz module:

```yaml
Strategies:
  MyStrategy:
    # ... other config ...
    Granter: "inj1granteraddress..."
    Grantees:
      - "inj1grantee1..."
      - "inj1grantee2..."
```

### Optional Components

Add external data sources or specialized components:

```python
# Add Redis/Valkey listener for external signals
client.add_component("ValkeyListener", {
    "ValkeyUrl": "valkey://localhost:6379",
    "ValkeyChannels": ["trading_signals", "market_data"]
})

# Add liquidation monitoring
client.add_component("HelixLiquidator")

# Add L3 orderbook streaming
client.add_component("HelixL3Streamer", {
    "MarketId": "0x17ef48032cb24375ba7c2e39f384e56433bcab20cbee9a7357e4cba2eb00abe6",
    "SleepInterval": 1
})
```

### Strategy Update Types

Your strategy can react to various market events:

- `UpdateType.OnOrderbook`: Orderbook changes
- `UpdateType.OnOraclePrice`: Oracle price updates
- `UpdateType.OnPosition`: Position changes
- `UpdateType.OnBankBalance`: Bank balance updates
- `UpdateType.OnDeposit`: Subaccount deposit changes
- `UpdateType.OnSpotOrder`: Spot order updates
- `UpdateType.OnDerivativeOrder`: Derivative order updates
- `UpdateType.OnSpotTrade`: Spot trade executions
- `UpdateType.OnDerivativeTrade`: Derivative trade executions
- `UpdateType.OnExternalInfo`: External data (Redis/Valkey)

### Risk Management

Configure risk parameters in your strategy:

```yaml
Strategies:
  MyStrategy:
    # ... other config ...
    RiskConfig:
      DrawdownWarning: 0.1
      DrawdownCritical: 0.2
      MarginWarning: 0.7
      MarginCritical: 0.8
```

### Performance Analytics

The framework includes built-in performance tracking:

```python
# Access performance metrics in your strategy
def check_performance(self):
    total_pnl = self.metrics.calculate_total_unrealized_pnl()
    win_rate = self.metrics.win_rate
    self.logger.info(f"Current PnL: {total_pnl}, Win Rate: {win_rate}%")
```

## API Reference

### TraderClient

Main client class for framework interaction:

```python
client = TraderClient(
    config_path="config.yaml",    # Path to configuration file
    network="mainnet",            # "mainnet" or "testnet"
    debug=False                   # Enable debug mode (no real transactions)
)

# Setup logging handlers if needed
from injective_trader.utils.logger import create_file_handler
file_handler = create_file_handler(log_file="logs/bot.log", level="DEBUG")
client.add_log_handler(file_handler)

# Add strategies
await client.add_strategy(strategy_name, strategy_class, strategy_config)

# Add optional components
client.add_component(component_name, config)

# Manual trading
await client.place_order(market_id, subaccount_id, side, price, quantity)
await client.cancel_order(market_id, subaccount_id, order_hash)

# Data access
markets = await client.get_markets()
accounts = await client.get_accounts()
positions = await client.get_positions()
orderbook = await client.get_orderbook(market_id)
```

### Strategy Base Class

```python
class Strategy(ABC):
    def on_initialize(self, accounts, markets) -> StrategyResult:
        """Called once during strategy initialization"""
        pass
        
    async def _execute_strategy(self, update_type, processed_data) -> StrategyResult:
        """Main strategy logic - implement your trading algorithm here"""
        pass
        
    def is_applicable(self, accounts, markets) -> bool:
        """Override to add custom activation conditions"""
        return True
```

## Configuration Reference

### Component Configuration

- **Initializer**: Market and account setup
- **ChainListener**: Real-time blockchain data streaming
- **MessageBroadcaster**: Transaction execution and batching
- **ValkeyListener**: External data integration (optional)
- **HelixLiquidator**: Liquidation monitoring (optional)
- **HelixL3Streamer**: Level 3 orderbook data (optional)

### Retry Configuration

Configure retry behavior for different components:

```yaml
RetryConfig:
  DefaultRetry:
    max_attempts: 3
    base_delay: 1.0
    max_delay: 32.0
    jitter: true
    timeout: 30.0
  ChainListener:
    max_attempts: 5
    timeout: 0  # No timeout for streaming
```

## Troubleshooting

### Common Issues

1. **Private Key Issues**: Ensure bot-specific private key environment variables are set (e.g., `MyBot_GRANTER_INJECTIVE_PRIVATE_KEY`)
2. **Network Connectivity**: Check your internet connection and Injective network status
3. **Market Not Found**: Verify market tickers in your configuration are valid
4. **Insufficient Funds**: Ensure accounts have sufficient balance for trading

### Debug Mode

Run with `--debug` flag to prevent actual transaction execution:

```bash
python main.py MyBot config.yaml logs/debug.log --debug
```

### Logging

Adjust logging levels in configuration:

```yaml
ConsoleLevel: INFO  # DEBUG, INFO, WARNING, ERROR, CRITICAL
FileLevel: DEBUG    # More verbose logging to file
```

## Examples

Check the repository for additional examples:
- Market making strategies
- Arbitrage strategies
- Grid trading strategies
- Risk management implementations

## Project Status

This project has an **Alpha release available** with **Beta currently in development**. We're actively developing and refining the framework with a focus on reliability, performance, and production readiness for the upcoming Beta release.

## Support

For questions and support:
- Review the [official Injective Trader documentation](https://docs.injective.network/developers-defi/injective-trader) and [examples](https://github.com/InjectiveLabs/injective-trader/tree/master/examples)
- Check existing issues in the repository
- Follow best practices for strategy development

## License

Copyright © 2021 - 2025 Injective Labs Inc. (https://injectivelabs.org/)

Originally released by Injective Labs Inc. under:
Apache License
Version 2.0, January 2004
http://www.apache.org/licenses/
