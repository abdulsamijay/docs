# Fuse

The Fuse platform enables anyone to instantly create their own lending and borrowing pool. Below you will find documentation for the Fuse platform. Each Fuse pool is essentially a fork of the Compound protocol.

## General

### Interpreting Exchange Rates

The fToken [Exchange Rate](https://compound.finance/docs/fTokens#exchange-rate) is scaled by the difference in decimals between the fToken and the underlying asset.

```js
const onefTokenInUnderlying =
  exchangeRateCurrent / ((1 * 10) ^ (18 + underlyingDecimals - fTokenDecimals));
```

Here is an example of finding the value of 1 fBAT in BAT with Web3.js JavaScript.

```js
const fTokenDecimals = 8; // all fTokens have 8 decimal places

const underlying = new web3.eth.Contract(erc20Abi, batAddress);

const fToken = new web3.eth.Contract(fTokenAbi, fBATAddress);

const underlyingDecimals = await underlying.methods.decimals().call();

const exchangeRateCurrent = await fToken.methods.exchangeRateCurrent().call();

const mantissa = 18 + parseInt(underlyingDecimals) - fTokenDecimals;

const onefTokenInUnderlying = exchangeRateCurrent / Math.pow(10, mantissa);

console.log("1 fBAT can be redeemed for", onefTokenInUnderlying, "BAT");
```

There is no underlying contract for ETH, so to do this with fETH, set underlyingDecimals to 18.

To find the number of underlying tokens that can be redeemed for fTokens, multiply the number of fTokens by the above value onefTokenInUnderlying.

```js
const underlyingTokens = fTokenAmount * onefTokenInUnderlying;
```

### Calculating Accrued Interest

Interest rates for each market update on any block in which the ratio of borrowed assets to supplied assets in the market has changed. The amount interest rates are changed depends on the interest rate model smart contract implemented for the market, and the amount of change in the ratio of borrowed assets to supplied assets in the market.

See the interest rate data visualization notebook on [Observable](https://observablehq.com/@jflatow/compound-interest-rates) to visualize which interest rate model is currently applied to each market.

Historical interest rates can be retrieved from the [MarketHistoryService API](https://compound.finance/docs/api#MarketHistoryService).

Interest accrues to all suppliers and borrowers in a market when any Ethereum address interacts with the market’s fToken contract, calling one of these functions: mint, redeem, borrow, or repay. Successful execution of one of these functions triggers the accrueInterest method, which causes interest to be added to the underlying balance of every supplier and borrower in the market. Interest accrues for the current block, as well as each prior block in which the accrueInterest method was not triggered (no user interacted with the fToken contract). Interest compounds only during blocks in which the fToken contract has one of the aforementioned methods invoked.

Here is an example of supply interest accrual:

Alice supplies 1 ETH to the Fuse protocol. At the time of supply, the supplyRatePerBlock is 37893605 Wei, or 0.000000000037893605 ETH per block. No one interacts with the fETHer contract for 3 Ethereum blocks. On the subsequent 4th block, Bob borrows some ETH. Alice’s underlying balance is now 1.000000000151574420 ETH (which is 37893605 Wei times 4 blocks, plus the original 1 ETH). Alice’s underlying ETH balance in subsequent blocks will have interest accrued based on the new value of 1.000000000151574420 ETH instead of the initial 1 ETH. Note that the supplyRatePerBlock value may change at any time.

### Calculating the APY Using Rate Per Block

The Annual Percentage Yield (APY) for supplying or borrowing in each market can be calculated using the value of supplyRatePerBlock (for supply APY) or borrowRatePerBlock (for borrow APY) in this formula:

```js
Rate = fToken.supplyRatePerBlock(); // Integer

Rate = 37893566

ETH Mantissa = 1 * 10 ^ 18 (ETH has 18 decimal places)

Blocks Per Day = 4 * 60 * 24 (based on 4 blocks occurring every minute)

Days Per Year = 365

APY = ((((Rate / ETH Mantissa * Blocks Per Day + 1) ^ Days Per Year)) - 1) * 100
```

Here is an example of calculating the supply and borrow APY with Web3.js JavaScript:

```js
const ethMantissa = 1e18;

const blocksPerDay = 4 * 60 * 24;

const daysPerYear = 365;

const fToken = new web3.eth.Contract(fETHAbi, fETHAddress);

const supplyRatePerBlock = await fToken.methods.supplyRatePerBlock().call();

const borrowRatePerBlock = await fToken.methods.borrowRatePerBlock().call();

const supplyApy =
  (Math.pow(
    (supplyRatePerBlock / ethMantissa) * blocksPerDay + 1,
    daysPerYear
  ) -
    1) *
  100;

const borrowApy =
  (Math.pow(
    (borrowRatePerBlock / ethMantissa) * blocksPerDay + 1,
    daysPerYear
  ) -
    1) *
  100;

console.log(`Supply APY for ETH ${supplyApy} %`);

console.log(`Borrow APY for ETH ${borrowApy} %`);
```

### Gas Costs

The gas usage of the protocol functions may fluctuate by market and user. External calls, such as to underlying ERC-20 tokens, may use an arbitrary amount of gas. Any calculations that involve checking [account liquidity](https://compound.finance/docs/comptroller#account-liquidity), have gas costs that increase with the number of [entered markets](https://compound.finance/docs/comptroller#enter-markets). Thus, while it can be difficult to provide any guarantees about costs, we provide the table below for guidance:

| Function         | Typical Gas Cost                     |
| ---------------- | ------------------------------------ |
| Mint             | < 150K, fDAI < 300k                  |
| Redeem, Transfer | < 250K if borrowing, otherwise < 90K |
| Borrow           | < 300K                               |
| Repay Borrow     | < 90K                                |
| Liquidate Borrow | < 400K                               |

## fToken(s)

### Mint

The mint function transfers an asset into the protocol, which begins accumulating interest based on the current [Supply Rate](https://compound.finance/docs/fTokens#supply-rate) for the asset. The user receives a quantity of fTokens equal to the underlying tokens supplied, divided by the current [Exchange Rate](https://compound.finance/docs/fTokens#exchange-rate).

#### fERC20

```solidity
function mint(uint mintAmount) returns (uint)
```

- `msg.sender`: The account which shall supply the asset, and own the minted fTokens.
- `mintAmount`: The amount of the asset to be supplied, in units of the underlying asset.
- `RETURN`: 0 on success, otherwise an [Error code](https://compound.finance/docs/fTokens#error-codes)

Before supplying an asset, users must first [approve](https://eips.ethereum.org/EIPS/eip-20#approve) the fToken to access their token balance.

#### fEther

```solidity
function mint() payable
```

- `msg.value`: The amount of ether to be supplied, in wei.
- `msg.sender`: The account which shall supply the ether, and own the minted fTokens.
- `RETURN`: No return, reverts on error.

#### Solidity

```solidity
Erc20 underlying = Erc20(0xToken...);     // get a handle for the underlying asset contract

fERC20 fToken = fERC20(0x3FDA...);        // get a handle for the corresponding fToken contract

underlying.approve(address(fToken), 100); // approve the transfer

assert(fToken.mint(100) == 0);            // mint the fTokens and assert there is no error
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

await fToken.methods.mint().send({from: myAccount, value: 50});
```

### Redeem

The redeem function converts a specified quantity of fTokens into the underlying asset, and returns them to the user. The amount of underlying tokens received is equal to the quantity of fTokens redeemed, multiplied by the current [Exchange Rate](https://compound.finance/docs/fTokens#exchange-rate). The amount redeemed must be less than the user's [Account Liquidity](https://compound.finance/docs/comptroller#account-liquidity) and the market's available liquidity.

#### fERC20 / fEther

```solidity
function redeem(uint redeemTokens) returns (uint)
```

- `msg.sender`: The account to which redeemed funds shall be transferred.
- `redeemTokens`: The number of fTokens to be redeemed.
- `RETURN`: 0 on success, otherwise an [Error code](https://compound.finance/docs/fTokens#error-codes)

#### Solidity

```solidity
fEther fToken = fEther(0x3FDB...);

require(fToken.redeem(7) == 0, "something went wrong");
```

#### Web3 1.0

```js
const fToken = fERC20.at(0x3FDA...);

fToken.methods.redeem(1).send({from: ...});
```

### Redeem Underlying

The redeem underlying function converts fTokens into a specified quantity of the underlying asset, and returns them to the user. The amount of fTokens redeemed is equal to the quantity of underlying tokens received, divided by the current [Exchange Rate](https://compound.finance/docs/fTokens#exchange-rate). The amount redeemed must be less than the user's [Account Liquidity](https://compound.finance/docs/comptroller#account-liquidity) and the market's available liquidity.

#### fERC20 / fEther

```solidity
function redeemUnderlying(uint redeemAmount) returns (uint)
```

- `msg.sender`: The account to which redeemed funds shall be transferred.
- `redeemAmount`: The amount of underlying to be redeemed.
- `RETURN`: 0 on success, otherwise an [Error code](https://compound.finance/docs/fTokens#error-codes)

#### Solidity

```solidity
fEther fToken = fEther(0x3FDB...);

require(fToken.redeemUnderlying(50) == 0, "something went wrong");
```

#### Web3 1.0

```js
const fToken = fERC20.at(0x3FDA...);

fToken.methods.redeemUnderlying(10).send({from: ...});
```

### Borrow

The borrow function transfers an asset from the protocol to the user, and creates a borrow balance which begins accumulating interest based on the [Borrow Rate](https://compound.finance/docs/fTokens#borrow-rate) for the asset. The amount borrowed must be less than the user's [Account Liquidity](https://compound.finance/docs/comptroller#account-liquidity) and the market's available liquidity.

To borrow Ether, the borrower must be 'payable' (solidity).

#### fERC20 / fEther

```solidity
function borrow(uint borrowAmount) returns (uint)
```

- `msg.sender`: The account to which borrowed funds shall be transferred.
- `borrowAmount` : The amount of the underlying asset to be borrowed.
- `RETURN`: 0 on success, otherwise an [Error code](https://compound.finance/docs/fTokens#error-codes)

#### Solidity

```solidity
fERC20 fToken = fERC20(0x3FDA...);

require(fToken.borrow(100) == 0, "got collateral?");
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

await fToken.methods.borrow(50).send({from: 0xMyAccount});
```

### Repay Borrow

The repay function transfers an asset into the protocol, reducing the user's borrow balance.

#### fERC20

```solidity
function repayBorrow(uint repayAmount) returns (uint)
```

- `msg.sender`: The account which borrowed the asset, and shall repay the borrow.
- `repayAmount`: The amount of the underlying borrowed asset to be repaid. A value of -1 (i.e. 2256 - 1) can be used to repay the full amount.
- `RETURN`: 0 on success, otherwise an [Error code](https://compound.finance/docs/fTokens#error-codes)

Before repaying an asset, users must first [approve](https://eips.ethereum.org/EIPS/eip-20#approve) the fToken to access their token balance.

#### fEther

```solidity
function repayBorrow() payable
```

- `msg.value`: The amount of ether to be repaid, in wei.
- `msg.sender`: The account which borrowed the asset, and shall repay the borrow.
- `RETURN`: No return, reverts on error.

#### Solidity

```solidity
fEther fToken = fEther(0x3FDB...);

require(fToken.repayBorrow.value(100)() == 0, "transfer approved?");
```

#### Web3 1.0

```js
const fToken = fERC20.at(0x3FDA...);

fToken.methods.repayBorrow(10000).send({from: ...});
```

### Repay Borrow Behalf

The repay function transfers an asset into the protocol, reducing the target user's borrow balance.

#### fERC20

```solidity
function repayBorrowBehalf(address borrower, uint repayAmount) returns (uint)
```

- `msg.sender`: The account which shall repay the borrow.
- `borrower`: The account which borrowed the asset to be repaid.
- `repayAmount`: The amount of the underlying borrowed asset to be repaid. A value of -1 (i.e. 2256 - 1) can be used to repay the full amount.
- `RETURN`: 0 on success, otherwise an [Error code](https://compound.finance/docs/fTokens#error-codes)

Before repaying an asset, users must first [approve](https://eips.ethereum.org/EIPS/eip-20#approve) the fToken to access their token balance.

#### fEther

```solidity
function repayBorrowBehalf(address borrower) payable
```

- `msg.value`: The amount of ether to be repaid, in wei.
- `msg.sender`: The account which shall repay the borrow.
- `borrower`: The account which borrowed the asset to be repaid.
- `RETURN`: No return, reverts on error.

#### Solidity

```solidity
fEther fToken = fEther(0x3FDB...);

require(fToken.repayBorrowBehalf.value(100)(0xBorrower) == 0, "transfer approved?");
```

#### Web3 1.0

```js
const fToken = fERC20.at(0x3FDA...);

await fToken.methods.repayBorrowBehalf(0xBorrower, 10000).send({from: 0xPayer});
```

### Liquidate Borrow

A user who has negative [account liquidity](https://compound.finance/docs/comptroller#account-liquidity) is subject to [liquidation](https://compound.finance/docs/fTokens#liquidate-borrow) by other users of the protocol to return his/her account liquidity back to positive (i.e. above the collateral requirement). When a liquidation occurs, a liquidator may repay some or all of an outstanding borrow on behalf of a borrower and in return receive a discounted amount of collateral held by the borrower; this discount is defined as the liquidation incentive.

A liquidator may close up to a certain fixed percentage (i.e. close factor) of any individual outstanding borrow of the underwater account. Unlike in v1, liquidators must interact with each fToken contract in which they wish to repay a borrow and seize another asset as collateral. When collateral is seized, the liquidator is transferred fTokens, which they may redeem the same as if they had supplied the asset themselves. Users must approve each fToken contract before calling liquidate (i.e. on the borrowed asset which they are repaying), as they are transferring funds into the contract.

```solidity
function liquidateBorrow(address borrower, uint amount, address collateral) returns (uint)
```

- `msg.sender`: The account which shall liquidate the borrower by repaying their debt and seizing their collateral.
- `borrower`: The account with negative account liquidity that shall be liquidated.
- `repayAmount`: The amount of the borrowed asset to be repaid and converted into collateral, specified in units of the underlying borrowed asset.
- `cTokenCollateral`: The address of the cToken currently held as collateral by a borrower, that the liquidator shall seize.
- `RETURN`: 0 on success, otherwise an Error code

### Transfer

Transfer is an ERC-20 method that allows accounts to send tokens to other Ethereum addresses. A fToken transfer will fail if the account has [entered](https://compound.finance/docs/comptroller#enter-markets) that fToken market and the transfer would have put the account into a state of negative [liquidity](https://compound.finance/docs/comptroller#account-liquidity).

#### fERC20 / fEther

```solidity
function transfer(address recipient, uint256 amount) returns (bool)
```

- `recipient`: The transfer recipient address.
- `amount`: The amount of fTokens to transfer.
- `RETURN`: Returns a boolean value indicating whether or not the operation succeeded.

#### Solidity

```solidity
fEther fToken = fEther(0x3FDB...);

fToken.transfer(0xABCD..., 100000000000);
```

#### Web3 1.0

```solidity
const fToken = fERC20.at(0x3FDA...);

await fToken.methods.transfer(0xABCD..., 100000000000).send({from: 0xSender});
```

#### fERC20

```solidity
function liquidateBorrow(address borrower, uint amount, address collateral) returns (uint)
```

- `msg.sender`: The account which shall liquidate the borrower by repaying their debt and seizing their collateral.
- `borrower`: The account with negative [account liquidity](https://compound.finance/docs/comptroller#account-liquidity) that shall be liquidated.
- `repayAmount`: The amount of the borrowed asset to be repaid and converted into collateral, specified in units of the underlying borrowed asset.
- `collateral`: The address of the fToken currently held as collateral by a borrower, that the liquidator shall seize.
- `RETURN`: 0 on success, otherwise an [Error code](https://compound.finance/docs/fTokens#error-codes)

Before supplying an asset, users must first [approve](https://eips.ethereum.org/EIPS/eip-20#approve) the fToken to access their token balance.

#### fEther

```solidity
function liquidateBorrow(address borrower, address fTokenCollateral) payable
```

- `msg.value`: The amount of ether to be repaid and converted into collateral, in wei.
- `msg.sender`: The account which shall liquidate the borrower by repaying their debt and seizing their collateral.
- `borrower`: The account with negative [account liquidity](https://compound.finance/docs/comptroller#account-liquidity) that shall be liquidated.
- `collateral`: The address of the fToken currently held as collateral by a borrower, that the liquidator shall seize.
- `RETURN`: No return, reverts on error.

#### Solidity

```solidity
fEther fToken = fEther(0x3FDB...);

fERC20 fTokenCollateral = fERC20(0x3FDA...);

require(fToken.liquidateBorrow.value(100)(0xBorrower, fTokenCollateral) == 0, "borrower underwater??");
```

#### Web3 1.0

```solidity
const fToken = fERC20.at(0x3FDA...);

const fTokenCollateral = fEther.at(0x3FDB...);

await fToken.methods.liquidateBorrow(0xBorrower, 33, fTokenCollateral).send({from: 0xLiquidator});
```

### Exchange Rate

Each fToken is convertible into an ever increasing quantity of the underlying asset, as interest accrues in the market. The exchange rate between a fToken and the underlying asset is equal to:

```solidity
exchangeRate = (getCash() + totalBorrows() - totalReserves()) / totalSupply()
```

#### fERC20 / fEther

```solidity
function exchangeRateCurrent() returns (uint)
```

- `RETURN`: The current exchange rate as an unsigned integer, scaled by 1e18.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint exchangeRateMantissa = fToken.exchangeRateCurrent();
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const exchangeRate = (await fToken.methods.exchangeRateCurrent().call()) / 1e18;
```

Tip: note the use of call vs. send to invoke the function from off-chain without incurring gas costs.

### Get Cash

Cash is the amount of underlying balance owned by this fToken contract. One may query the total amount of cash currently available to this market.

#### fERC20 / fEther

```solidity
function getCash() returns (uint)
```

- `RETURN`: The quantity of underlying asset owned by the contract.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint cash = fToken.getCash();
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const cash = (await fToken.methods.getCash().call());
```

### Total Borrow

Total Borrows is the amount of underlying currently loaned out by the market, and the amount upon which interest is accumulated to suppliers of the market.

#### fERC20 / fEther

```solidity
function totalBorrowsCurrent() returns (uint)
```

- `RETURN`: The total amount of borrowed underlying, with interest.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint borrows = fToken.totalBorrowsCurrent();
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const borrows = (await fToken.methods.totalBorrowsCurrent().call());
```

### Borrow Balance

A user who borrows assets from the protocol is subject to accumulated interest based on the current [borrow rate](https://compound.finance/docs/fTokens#borrow-rate). Interest is accumulated every block and integrations may use this function to obtain the current value of a user's borrow balance with interest.

#### fERC20 / fEther

```solidity
function borrowBalanceCurrent(address account) returns (uint)
```

- `account`: The account which borrowed the assets.
- `RETURN`: The user's current borrow balance (with interest) in units of the underlying asset.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint borrows = fToken.borrowBalanceCurrent(msg.caller);
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const borrows = await fToken.methods.borrowBalanceCurrent(account).call();
```

### Borrow Rate

At any point in time one may query the contract to get the current borrow rate per block.

#### fERC20 / fEther

```solidity
function borrowRatePerBlock() returns (uint)
```

- `RETURN`: The current borrow rate as an unsigned integer, scaled by 1e18.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint borrowRateMantissa = fToken.borrowRatePerBlock();
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const borrowRate = (await fToken.methods.borrowRatePerBlock().call()) / 1e18;
```

### Total Supply

Total Supply is the number of tokens currently in circulation in this fToken market. It is part of the EIP-20 interface of the fToken contract.

#### fERC20 / fEther

```solidity
function totalSupply() returns (uint)
```

- `RETURN`: The total number of tokens in circulation for the market.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint tokens = fToken.totalSupply();
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const tokens = (await fToken.methods.totalSupply().call());
```

### Underlying Balance

The user's underlying balance, representing their assets in the protocol, is equal to the user's fToken balance multiplied by the [Exchange Rate](https://compound.finance/docs/fTokens#exchange-rate).

#### fERC20 / fEther

```solidity
function balanceOfUnderlying(address account) returns (uint)
```

- `account`: The account to get the underlying balance of.
- `RETURN`: The amount of underlying currently owned by the account.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint tokens = fToken.balanceOfUnderlying(msg.caller);
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const tokens = await fToken.methods.balanceOfUnderlying(account).call();
```

### Supply Rate

At any point in time one may query the contract to get the current supply rate per block. The supply rate is derived from the [borrow rate](https://compound.finance/docs/fTokens#borrow-rate), [reserve factor](https://compound.finance/docs/fTokens#reserve-factor) and the amount of [total borrows](https://compound.finance/docs/fTokens#total-borrows).

#### fERC20 / fEther

```solidity
function supplyRatePerBlock() returns (uint)
```

- `RETURN`: The current supply rate as an unsigned integer, scaled by 1e18.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint supplyRateMantissa = fToken.supplyRatePerBlock();
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const supplyRate = (await fToken.methods.supplyRatePerBlock().call()) / 1e18;
```

### Total Reserves

Reserves are an accounting entry in each fToken contract that represents a portion of historical interest set aside as [cash](https://compound.finance/docs/fTokens#cash) which can be withdrawn or transferred through the protocol's governance. A small portion of borrower interest accrues into the protocol, determined by the [reserve factor](https://compound.finance/docs/fTokens#reserve-factor).

#### fERC20 / fEther

```solidity
function totalReserves() returns (uint)
```

- `RETURN`: The total amount of reserves held in the market.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint reserves = fToken.totalReserves();
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const reserves = (await fToken.methods.totalReserves().call());
```

### Reserve Factor

The reserve factor defines the portion of borrower interest that is converted into [reserves](https://compound.finance/docs/fTokens#total-reserves).

#### fERC20 / fEther

```solidity
function reserveFactorMantissa() returns (uint)
```

- `RETURN`: The current reserve factor as an unsigned integer, scaled by 1e18.

#### Solidity

```solidity
fERC20 fToken = fToken(0x3FDA...);

uint reserveFactorMantissa = fToken.reserveFactorMantissa();
```

#### Web3 1.0

```js
const fToken = fEther.at(0x3FDB...);

const reserveFactor = (await fToken.methods.reserveFactorMantissa().call()) / 1e18;
```

### Key Events

| Event                                                                                                               | Description                                                                                           |
| ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Mint(address minter, uint mintAmount, uint mintTokens)                                                              | Emitted upon a successful [Mint](https://compound.finance/docs/fTokens#mint).                         |
| Redeem(address redeemer, uint redeemAmount, uint redeemTokens)                                                      | Emitted upon a successful [Redeem](https://compound.finance/docs/fTokens#redeem).                     |
| Borrow(address borrower, uint borrowAmount, uint accountBorrows, uint totalBorrows)                                 | Emitted upon a successful [Borrow](https://compound.finance/docs/fTokens#borrow).                     |
| RepayBorrow(address payer, address borrower, uint repayAmount, uint accountBorrows, uint totalBorrows)              | Emitted upon a successful [Repay Borrow](https://compound.finance/docs/fTokens#repay-borrow).         |
| LiquidateBorrow(address liquidator, address borrower, uint repayAmount, address fTokenCollateral, uint seizeTokens) | Emitted upon a successful [Liquidate Borrow](https://compound.finance/docs/fTokens#liquidate-borrow). |

### Error Codes

| Code | Name                           | Description                                                                                                                                                                    |
| ---- | ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 0    | NO_ERROR                       | Not a failure.                                                                                                                                                                 |
| 1    | UNAUTHORIZED                   | The sender is not authorized to perform this action.                                                                                                                           |
| 2    | BAD_INPUT                      | An invalid argument was supplied by the caller.                                                                                                                                |
| 3    | COMPTROLLER_REJECTION          | The action would violate the comptroller policy.                                                                                                                               |
| 4    | COMPTROLLER_CALCULATION_ERROR  | An internal calculation has failed in the comptroller.                                                                                                                         |
| 5    | INTEREST_RATE_MODEL_ERROR      | The interest rate model returned an invalid value.                                                                                                                             |
| 6    | INVALID_ACCOUNT_PAIR           | The specified combination of accounts is invalid.                                                                                                                              |
| 7    | INVALID_CLOSE_AMOUNT_REQUESTED | The amount to liquidate is invalid.                                                                                                                                            |
| 8    | INVALID_COLLATERAL_FACTOR      | The collateral factor is invalid.                                                                                                                                              |
| 9    | MATH_ERROR                     | A math calculation error occurred.                                                                                                                                             |
| 10   | MARKET_NOT_FRESH               | Interest has not been properly accrued.                                                                                                                                        |
| 11   | MARKET_NOT_LISTED              | The market is not currently listed by its comptroller.                                                                                                                         |
| 12   | TOKEN_INSUFFICIENT_ALLOWANCE   | ERC-20 contract must _allow_ Money Market contract to call transferFrom. The current allowance is either 0 or less than the requested supply, repayBorrow or liquidate amount. |
| 13   | TOKEN_INSUFFICIENT_BALANCE     | Caller does not have sufficient balance in the ERC-20 contract to complete the desired action.                                                                                 |
| 14   | TOKEN_INSUFFICIENT_CASH        | The market does not have a sufficient cash balance to complete the transaction. You may attempt this transaction again later.                                                  |
| 15   | TOKEN_TRANSFER_IN_FAILED       | Failure in ERC-20 when transfering token into the market.                                                                                                                      |
| 16   | TOKEN_TRANSFER_OUT_FAILED      | Failure in ERC-20 when transfering token out of the market.                                                                                                                    |

### Failure Info

| Code | Name                                                       |
| ---- | ---------------------------------------------------------- |
| 0    | ACCEPT_ADMIN_PENDING_ADMIN_CHECK                           |
| 1    | ACCRUE_INTEREST_ACCUMULATED_INTEREST_CALCULATION_FAILED    |
| 2    | ACCRUE_INTEREST_BORROW_RATE_CALCULATION_FAILED             |
| 3    | ACCRUE_INTEREST_NEW_BORROW_INDEX_CALCULATION_FAILED        |
| 4    | ACCRUE_INTEREST_NEW_TOTAL_BORROWS_CALCULATION_FAILED       |
| 5    | ACCRUE_INTEREST_NEW_TOTAL_RESERVES_CALCULATION_FAILED      |
| 6    | ACCRUE_INTEREST_SIMPLE_INTEREST_FACTOR_CALCULATION_FAILED  |
| 7    | BORROW_ACCUMULATED_BALANCE_CALCULATION_FAILED              |
| 8    | BORROW_ACCRUE_INTEREST_FAILED                              |
| 9    | BORROW_CASH_NOT_AVAILABLE                                  |
| 10   | BORROW_FRESHNESS_CHECK                                     |
| 11   | BORROW_NEW_TOTAL_BALANCE_CALCULATION_FAILED                |
| 12   | BORROW_NEW_ACCOUNT_BORROW_BALANCE_CALCULATION_FAILED       |
| 13   | BORROW_MARKET_NOT_LISTED                                   |
| 14   | BORROW_COMPTROLLER_REJECTION                               |
| 15   | LIQUIDATE_ACCRUE_BORROW_INTEREST_FAILED                    |
| 16   | LIQUIDATE_ACCRUE_COLLATERAL_INTEREST_FAILED                |
| 17   | LIQUIDATE_COLLATERAL_FRESHNESS_CHECK                       |
| 18   | LIQUIDATE_COMPTROLLER_REJECTION                            |
| 19   | LIQUIDATE_COMPTROLLER_CALCULATE_AMOUNT_SEIZE_FAILED        |
| 20   | LIQUIDATE_CLOSE_AMOUNT_IS_UINT_MAX                         |
| 21   | LIQUIDATE_CLOSE_AMOUNT_IS_ZERO                             |
| 22   | LIQUIDATE_FRESHNESS_CHECK                                  |
| 23   | LIQUIDATE_LIQUIDATOR_IS_BORROWER                           |
| 24   | LIQUIDATE_REPAY_BORROW_FRESH_FAILED                        |
| 25   | LIQUIDATE_SEIZE_BALANCE_INCREMENT_FAILED                   |
| 26   | LIQUIDATE_SEIZE_BALANCE_DECREMENT_FAILED                   |
| 27   | LIQUIDATE_SEIZE_COMPTROLLER_REJECTION                      |
| 28   | LIQUIDATE_SEIZE_LIQUIDATOR_IS_BORROWER                     |
| 29   | LIQUIDATE_SEIZE_TOO_MUCH                                   |
| 30   | MINT_ACCRUE_INTEREST_FAILED                                |
| 31   | MINT_COMPTROLLER_REJECTION                                 |
| 32   | MINT_EXCHANGE_CALCULATION_FAILED                           |
| 33   | MINT_EXCHANGE_RATE_READ_FAILED                             |
| 34   | MINT_FRESHNESS_CHECK                                       |
| 35   | MINT_NEW_ACCOUNT_BALANCE_CALCULATION_FAILED                |
| 36   | MINT_NEW_TOTAL_SUPPLY_CALCULATION_FAILED                   |
| 37   | MINT_TRANSFER_IN_FAILED                                    |
| 38   | MINT_TRANSFER_IN_NOT_POSSIBLE                              |
| 39   | REDEEM_ACCRUE_INTEREST_FAILED                              |
| 40   | REDEEM_COMPTROLLER_REJECTION                               |
| 41   | REDEEM_EXCHANGE_TOKENS_CALCULATION_FAILED                  |
| 42   | REDEEM_EXCHANGE_AMOUNT_CALCULATION_FAILED                  |
| 43   | REDEEM_EXCHANGE_RATE_READ_FAILED                           |
| 44   | REDEEM_FRESHNESS_CHECK                                     |
| 45   | REDEEM_NEW_ACCOUNT_BALANCE_CALCULATION_FAILED              |
| 46   | REDEEM_NEW_TOTAL_SUPPLY_CALCULATION_FAILED                 |
| 47   | REDEEM_TRANSFER_OUT_NOT_POSSIBLE                           |
| 48   | REDUCE_RESERVES_ACCRUE_INTEREST_FAILED                     |
| 49   | REDUCE_RESERVES_ADMIN_CHECK                                |
| 50   | REDUCE_RESERVES_CASH_NOT_AVAILABLE                         |
| 51   | REDUCE_RESERVES_FRESH_CHECK                                |
| 52   | REDUCE_RESERVES_VALIDATION                                 |
| 53   | REPAY_BEHALF_ACCRUE_INTEREST_FAILED                        |
| 54   | REPAY_BORROW_ACCRUE_INTEREST_FAILED                        |
| 55   | REPAY_BORROW_ACCUMULATED_BALANCE_CALCULATION_FAILED        |
| 56   | REPAY_BORROW_COMPTROLLER_REJECTION                         |
| 57   | REPAY_BORROW_FRESHNESS_CHECK                               |
| 58   | REPAY_BORROW_NEW_ACCOUNT_BORROW_BALANCE_CALCULATION_FAILED |
| 59   | REPAY_BORROW_NEW_TOTAL_BALANCE_CALCULATION_FAILED          |
| 60   | REPAY_BORROW_TRANSFER_IN_NOT_POSSIBLE                      |
| 61   | SET_COLLATERAL_FACTOR_OWNER_CHECK                          |
| 62   | SET_COLLATERAL_FACTOR_VALIDATION                           |
| 63   | SET_COMPTROLLER_OWNER_CHECK                                |
| 64   | SET_INTEREST_RATE_MODEL_ACCRUE_INTEREST_FAILED             |
| 65   | SET_INTEREST_RATE_MODEL_FRESH_CHECK                        |
| 66   | SET_INTEREST_RATE_MODEL_OWNER_CHECK                        |
| 67   | SET_MAX_ASSETS_OWNER_CHECK                                 |
| 68   | SET_ORACLE_MARKET_NOT_LISTED                               |
| 69   | SET_PENDING_ADMIN_OWNER_CHECK                              |
| 70   | SET_RESERVE_FACTOR_ACCRUE_INTEREST_FAILED                  |
| 71   | SET_RESERVE_FACTOR_ADMIN_CHECK                             |
| 72   | SET_RESERVE_FACTOR_FRESH_CHECK                             |
| 73   | SET_RESERVE_FACTOR_BOUNDS_CHECK                            |
| 74   | TRANSFER_COMPTROLLER_REJECTION                             |
| 75   | TRANSFER_NOT_ALLOWED                                       |
| 76   | TRANSFER_NOT_ENOUGH                                        |
| 77   | TRANSFER_TOO_MUCH                                          |