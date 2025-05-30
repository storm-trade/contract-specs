# Storm Protocol TL-B Structures

This document describes the core data structures and message types used in the Storm protocol on TON blockchain. Storm is a decentralized perpetual futures protocol that allows trading with leverage and various order types.

## Basic Types

### Direction
- `long$0` - Long position direction (buying with expectation of price increase)
- `short$1` - Short position direction (selling with expectation of price decrease)

### Position Data
```tlb
position#_ size:int128 direction:Direction margin:Coins open_notional:Coins last_updated_cumulative_premium:int64 fee:uint32 discount:uint32 rebate:uint32 last_updated_timestamp:uint32
```
- `size` - Position size in base asset units (e.g., BTC, ETH). Positive for long positions, negative for short positions
- `direction` - Position direction (long/short)
- `margin` - Collateral amount that backs the position
- `open_notional` - Position value in quote asset.
- `last_updated_cumulative_premium` - Last updated cumulative funding rate. Used to track funding payments between long and short positions
- `fee` - Trading fee in percents
- `discount` - Fee discount in percents
- `rebate` - Percentage of total trading fees that will be distributed to the user's referrer (if the user has one)
- `last_updated_timestamp` - Last update timestamp of the position data

### AMM State Log
```tlb
amm_state_log#_ quote_asset_reserve:Coins quote_asset_weight:Coins base_asset_reserve:Coins total_long_position_size:Coins total_short_position_size:Coins open_interest_long:Coins open_interest_short:Coins
```

> **Note**: The following fields are deprecated and were used in previous versions of the protocol. They are maintained for backward compatibility:
> - `quote_asset_reserve` - Virtual reserve of quote asset (previously used in AMM calculations)
> - `quote_asset_weight` - Virtual weight of quote asset (previously used for price impact calculations)
> - `base_asset_reserve` - Virtual reserve of base asset (previously used in AMM calculations)

Current active fields:
- `total_long_position_size` - Total size of all long positions in the market
- `total_short_position_size` - Total size of all short positions in the market
- `open_interest_long` - Total open interest for long positions (total value of all long positions)
- `open_interest_short` - Total open interest for short positions (total value of all short positions)

### Market Depth
```tlb
market_depth#_ vpi_spread:Coins vpi_market_depth_long:Coins vpi_market_depth_short:Coins vpi_coefficient:uint64
```
- `vpi_spread` - Current spread between long and short positions. Higher spread indicates lower liquidity
- `vpi_market_depth_long` - Amount of liquidity required to move the price up by 2%
- `vpi_market_depth_short` - Amount of liquidity required to move the price down by 2%
- `vpi_coefficient` - Market depth coefficient used to calculate price impact based on order size (constant)

## Order Types

### Stop Loss Order
```tlb
stop_loss_order$0000 expiration:uint32 direction:Direction amount:Coins trigger_price:Coins
```
- `expiration` - Order expiration timestamp. After this time, the order is automatically cancelled
- `direction` - Order direction
- `amount` - Order size in base asset units
- `trigger_price` - Price at which the order will be executed. For long positions, triggers when price falls below this level. For short positions, triggers when price rises above this level

### Take Profit Order
```tlb
take_profit_order$0001 expiration:uint32 direction:Direction amount:Coins trigger_price:Coins
```
- `expiration` - Order expiration timestamp
- `direction` - Order direction
- `amount` - Order size in base asset units
- `trigger_price` - Price at which the order will be executed. For long positions, triggers when price rises above this level. For short positions, triggers when price falls below this level

### Limit Order
```tlb
stop_limit_order$0010 expiration:uint32 direction:Direction amount:Coins leverage:uint64 limit_price:Coins stop_price:Coins stop_trigger_price:Coins take_trigger_price:Coins
```
- `expiration` - Order expiration timestamp
- `direction` - Order direction
- `amount` - Order size in quote asset units
- `leverage` - Position leverage (e.g., 2x, 5x, 10x)
- `limit_price` - Price at which the limit order will be executed
- `stop_price` - Minimum amount to receive in base asset (slippage protection). Order will be rejected if received amount is less than this
- `stop_trigger_price` - Optional stop loss price that will be automatically set for the position if order is executed
- `take_trigger_price` - Optional take profit price that will be automatically set for the position if order is executed

### Market Order
```tlb
market_order$0011 expiration:uint32 direction:Direction amount:Coins leverage:uint64 limit_price:Coins stop_price:Coins stop_trigger_price:Coins take_trigger_price:Coins
```
- `expiration` - Order expiration timestamp
- `direction` - Order direction
- `amount` - Order size in quote asset units
- `leverage` - Position leverage
- `limit_price` - Maximum/minimum execution price to prevent slippage
- `stop_price` - Minimum amount to receive in base asset (slippage protection). Order will be rejected if received amount is less than this
- `stop_trigger_price` - Optional stop loss price that will be automatically set for the position if order is executed
- `take_trigger_price` - Optional take profit price that will be automatically set for the position if order is executed

## Oracle Messages

### Update Message
```tlb
update_msg#_ price:Coins spread:Coins timestamp:uint32 assetIndex:uint16 pause_at:uint32 unpause_at:uint32 vpi_spread:Coins vpi_market_depth_long:Coins vpi_market_depth_short:Coins vpi_coefficient:uint64
```
- `price` - Current asset price from the oracle
- `spread` - Maximum allowed price deviation from the oracle price. Used to prevent price manipulation and ensure fair execution
- `timestamp` - Update timestamp of the oracle price
- `assetIndex` - Asset identifier
- `pause_at` - Timestamp when trading will be paused (e.g., for maintenance, forex, equity and stocks)
- `unpause_at` - Timestamp when trading will be resumed after pause
- `vpi_spread`, `vpi_market_depth_long`, `vpi_market_depth_short`, `vpi_coefficient` - See [Market Depth](#market-depth) section for detailed description of these fields

### Oracle Payload Types
The protocol supports different types of oracle updates to handle various market scenarios:

- `single#00` - Single oracle update with one price point. Used for normal market conditions
- `double#01` - Double oracle update with settlement price. Used when:
  - Market uses non-USDT collateral (e.g., TON). Settlement price allows converting collateral value to USD equivalent
  - For example, if 1 TON = $2.5, and user has 100 TON as collateral, settlement price helps calculate USD value ($250)
  - This mechanism allows the protocol to operate with any collateral while maintaining USD-denominated positions
- `single_with_created#02` - Single update with created price. Used for guaranteed price execution:
  - When an order's trigger price falls between two oracle updates, protocol assumes all prices in that interval existed
  - For example, if current price is $2.3, user sets stop/take at $2.4, and next price is $2.5, protocol uses created price $2.4
  - This ensures fair execution of stop-loss and take-profit orders even when price moves quickly
  - Helps prevent price manipulation and ensures orders are executed at their intended prices
- `double_with_created#03` - Double update with created price and settlement. Combines both mechanisms:
  - Used when both guaranteed price execution and collateral conversion are needed
  - For example, when executing orders during funding period with non-USDT collateral
  - Allows precise position updates while maintaining USD-denominated calculations

## Internal Messages

### Update Position
```tlb
update_position#60dfc677 direction:Direction origin_op:uint32 oracle_price:Coins settlement_oracle_price:Coins new_position_ref:^PositionData amm_state_log:^AmmStateLog
```
- `direction` - Position direction
- `origin_op` - Original operation ID for tracking the update
- `oracle_price` - Current oracle price at the time of update
- `settlement_oracle_price` - Price used to convert collateral value to USD equivalent. For example:
  - If collateral is TON and 1 TON = $2.5, this price helps calculate USD value of positions
  - Allows protocol to maintain USD-denominated positions while using any collateral
  - Used for funding calculations, liquidation checks, and position value updates
- `new_position_ref` - Reference to updated position data after the change
- `amm_state_log` - Reference to updated AMM state after the position update

### Complete Order
```tlb
complete_order#cf90d618 order_type:uint4 order_index:uint3 direction:Direction origin_op:uint32 oracle_price:Coins settlement_oracle_price:Coins new_position_ref:^PositionData amm_state_log:^AmmStateLog market_depth_log_ref:^MarketDepth
```
- `order_type` - Type of the order (0-15):
  - 0: Stop Loss
  - 1: Take Profit
  - 2: Limit
  - 3: Market Order
- `order_index` - Order index (0-7) in the order book
- `direction` - Order direction
- `origin_op` - Original operation ID
- `oracle_price` - Current oracle price at execution
- `settlement_oracle_price` - Price used to convert collateral value to USD equivalent:
  - Essential for non-USDT collateral to maintain USD-denominated position calculations
  - Used in PnL calculations and position updates
- `new_position_ref` - Reference to updated position data after order execution
- `amm_state_log` - Reference to updated AMM state after order execution
- `market_depth_log_ref` - Reference to updated market depth data

### Trade Notification
```tlb
trade_notification#3475fdd2 asset_id:uint16 free_amount:int64 locked_amount:int64 exchange_amount:int64 withdraw_locked_amount:uint64 fee_to_stakers:uint64 withdraw_amount:uint64 trader_addr:MsgAddressInt origin_addr:MsgAddressInt referral_data:(Maybe ^NotificationReferralParams) executor_params:(Maybe ^NotificationExecutorParams)
```
- `asset_id` - Asset identifier
- `free_amount` - Amount of tokens in the liquidity provider pool
- `locked_amount` - Total amount of tokens used as collateral for all trading positions
- `exchange_amount` - Amount of tokens exchanged between free and locked tokens
- `withdraw_locked_amount` - Amount of tokens to be withdrawn from the locked portion (collateral)
- `fee_to_stakers` - Fee amount distributed to protocol stakers
- `withdraw_amount` - Total amount of tokens to be withdrawn (sum of withdrawals from both free and locked portions)
- `trader_addr` - Trader's address
- `origin_addr`
- `referral_data` - Optional referral parameters for referral program:
  - `referral_amount` - Amount of tokens allocated for referral reward
  - `referral_addr` - Address of the referrer's NFT
- `executor_params` - Optional executor parameters for delegated trading:
  - `split_executor_reward` - Flag indicating if executor reward should be split
  - `executor_amount` - Amount of tokens allocated for executor reward
  - `executor_index` - Index of the executor in the executor list

### Execute Order
```tlb
execute_order#de1ddbcc direction:Direction order_index:uint3 trader_addr:MsgAddressInt prev_addr:MsgAddressInt ref_addr:MsgAddress executor_index:uint32 order:^Order position:^PositionData oracle_payload:^OraclePayload
```
- `direction` - Order direction
- `order_index` - Order index (0-7) in the order book
- `trader_addr` - Address of the trader who placed the order
- `prev_addr` - Sender address for returning remaining gas
- `ref_addr` - Referrer's NFT address
- `executor_index` - Index of the executor who will execute the order
- `order` - Reference to the order data
- `position` - Reference to the current position data
- `oracle_payload` - Reference to the oracle data used for execution 