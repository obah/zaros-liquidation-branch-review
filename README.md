# zaros-liquidation-branch-review

Smart Contract Review
Smart Contract: Zaros Finance LiquidationBranch.sol
Link to repo: https://github.com/Cyfrin/2024-07-zaros/blob/main/src/perpetuals/branches/LiquidationBranch.sol

Introduction: Zaros Finance is a perpetuals decentralised exchanged that uses restaking vaults to improve their user’s LST and LRT yield. It is built on Arbitrum and Monad. The LiquidationBranch.sol is used to check liquidable account and liquidate accounts in Zaros Finance.

Contract

pragma solidity 0.8.25: the compiler version used to compile the contract by the EVM Compiler.

Imports Overview:
`import { Errors } from "@zaros/utils/Errors.sol";`
- Errors.sol – This is a library that holds all the custom errors being used around the entire repo, more like for easy reuse and not recreating the errors in every single contract or library or .sol that needs it.

`import { FeeRecipients } from "@zaros/perpetuals/leaves/FeeRecipients.sol";`
- FeeRecipients.sol – This is a library that holds a single struct called Data and the Data struct contains 3 addresses, which are the receivers of the liquidated collateral, order fee and settlement fee which are used in several places in this contract.

`import { GlobalConfiguration } from "@zaros/perpetuals/leaves/GlobalConfiguration.sol";`
- GlobalConfiguration.sol – This is a library containing a bytes32 constant GLOBAL_CONFIGURATION_LOCATION, struct Data and several functions all allowing the owner to configure and change the available markets and collateral liquidation priority configuration. This library imports and uses EnumerableSet and SafeCast from OpenZeppelin and the Errors.sol library.  The LiquidationBranch contract uses the Data struct and some of the functions from GlobalConfiguration.

`import { TradingAccount } from "@zaros/perpetuals/leaves/TradingAccount.sol";`
- TradingAccount.sol – This is a library containing a bytes32 constant TRADING_ACCOUNT_LOCATION, a struct Data and several functions meant to control and configure different aspects of a trade entered by an account such as the create new positions, active positions, margin requirements, equity etc This library imports several Zaros dependencies such as Errors, FeeRecipients etc, some libraries from OpenZeppelin like SafeCast, EnumberableSet and PRB Math dependecies UD60x18 and SD59X18. The data struct and some other functions from this library are used in the  LiquidationBranch smart contract.

`import { PerpMarket } from "@zaros/perpetuals/leaves/PerpMarket.sol";`
- PerpMarket.sol – This is a library containing a bytes32 constant PERP_MARKET_LOCATION, a Data struct and several functions, all used to handle a market, such as getting index price, mark price, current funding rate, current funding velocity, skew etc. This library imports several Zaros dependencies such as Errors, FeeRecipients etc, some libraries from OpenZeppelin like SafeCast, EnumberableSet and PRB Math dependencies UD60x18 and SD59X18. Functions from this library are used in the liquidateAccount() function of the LiquidationBranch smart contract.

`import { Position } from "@zaros/perpetuals/leaves/Position.sol";`
- Position.sol – This library contains a bytes32 constant POSITION_LOCATION, struct Data, struct State and different functions to get information on a position, such as a margin requirement, accrued funding, notional value, unrealized pnl etc. This library has 2 imports which are PRB Math dependecies UD60x18 and SD59X18 which are used in State struct. The Data struct and other functions from this library are used in different areas of the LiquidationBranch smart contract.

`import { MarketOrder } from "@zaros/perpetuals/leaves/MarketOrder.sol";`
- MarketOrder.sol – This is a library that handles market orders. It has a bytes32 constant MARKET_ORDER_LOCATION, a struct Data and functions load(), loadExisting(), update(), clear() and checkPendingOrder(). The Data struct and load() function is used in the liquidateAccount() function of the LiquidationBranch smart contract.

`import { EnumerableSet } from "@openzeppelin/utils/structs/EnumerableSet.sol";`
- EnumerableSet.sol – This is a library from OpenZeppelin. It provides properties of a set with the ability to enumerate the items in it, thereby enabling efficient iteration. The LiquidationBranch contract uses the remove function from the EnumerableSet library when updating active markets in the TradingAccount.sol smart contract.

`import { SafeCast } from "@openzeppelin/utils/math/SafeCast.sol";`
- SafeCast.sol – Another library from OpenZeppelin. It provides functionality to safely downcast larger integer types to smaller integer types, like uint256 to uint128 and avoid the potential overflow and data loss risks.  The LiquidationBranch contract uses the safecast functionality for its uint256.

`import { UD60x18, ud60x18, ZERO as UD60x18_ZERO } from "@prb-math/UD60x18.sol";`
- UD60x18.sol - This library provides tools for high-precision arithmetics with unsigned fixed-point numbers in solidity. It provides operations for numbers that use 60.18 decimal fixed-point notation. 3 packages are imported from this contract:

- UD60x18 – The UD60x18 is the core type defined in the UD60x18.sol library. It represents an unsigned decimal number with 18 decimal places in solidity’s 256-bit word size, meaning a uint256 scaled by 10**18 to accommodate 18 decimal places.

- ud60x18 – This is the constructor of the UD60x18.sol library. It takes a uint256 which is scaled to 10**18 i.e the UD60x18 type.

- ZERO – This is a constant defined in  UD60x18.sol library representing 0 in UD60x18 type. Due to another import being called ZERO, an alias is used to rename it as UD60x18_ZERO using the ‘as’ keyword.

The 3 packages are used in different functions and type declarations in the LiquidationContext struct of the  LiquidationBranch smart contract.

`import { SD59x18, sd59x18, ZERO as SD59x18_ZERO } from "@prb-math/SD59x18.sol";`
- SD59x18.sol - This library provides tools for high-precision arithmetics with signed fixed-point numbers in solidity. It provides operations for numbers that use 59.18 decimal fixed point notation. 3 packages are imported from this contract:

- SD59x18 – The SD59x18 is the core type defined in the SD59x18.sol library. It represents signed decimal number with 59 bits for the integer part and 18 bits for the decimal part, scaling a uint256  by 10**18 to accommodate 18 decimal places.

- sd59x18 – This is the constructor of the SD59x18.sol library. It takes a int256 which is scaled to 10**18 i.e the SD59x18 type.

- ZERO – This is a constant defined in  SD59x18.sol library representing 0 in SD59x18 type. Due to another import being called ZERO, an alias is used to rename it as SD59x18_ZERO using the ‘as’ keyword.

The 3 packages are used in different functions and type declarations in the LiquidationContext struct of the  LiquidationBranch smart contract.


CONTRACT STRUCTURE DEFINITION
`contract LiquidationBranch {`
- This defines the start of the contract scope.

`using EnumerableSet for EnumerableSet.UintSet;`
- The EnumerableSet library is being initialized here and used on UintSet which is a struct from the EnumberableSet library. With this declaration, functions like add(), remove(), contains() from EnumerableSet can be called directly on type UintSet when it’s used in the LiquidationBranch smart contract.

`using GlobalConfiguration for GlobalConfiguration.Data;`
- The  GlobalConfiguration library is initialized and attached to the Data struct from the  GlobalConfiguration library. This makes functions like addMarket(), removeMarket(), and checkMarketIsEnabled() which are all available in the GlobalConfiguration library available to the Data struct when it’s used in the LiquidationBranch smart contract. 
 
`using TradingAccount for TradingAccount.Data;`
- The TradingAccount library is being initialized and attached to the Data struct from the TradingAccount library. This makes functions like loadExistingAccountAndVerifySender(),  validatePositionsLimit(), and validateMarginRequirement() which are from the TradingAccount library available to the Data struct when it’s used in the  LiquidationBranch smart contract.

`using PerpMarket for PerpMarket.Data;`
- The PerpMarket library is being initialized and attached to the Data struct from the PerpMarket library. This makes functions like getIndexPrice(),  getMarkPrice(), and getCurrentFundingRate() which are from the PerpMarket library available to the Data struct when it’s used in the LiquidationBranch smart contract.

`using Position for Position.Data;`
- The Position library is being initialized and attached to the Data struct from  Position library. This makes functions like getState(),  getAccruedFunding(), update() which are from Position library available to the Data struct when it’s used in the LiquidationBranch smart contract.

`using MarketOrder for MarketOrder.Data;`
- the Order library is being initialized and attached to the Data struct from the MarketOrder library. This makes functions like checkPendingOrder(),  update(), clear() which are from the MarketOrder library available to the Data struct when it’s used in the LiquidationBranch smart contract.

`using SafeCast for uint256;`
- The SafeCast library is initialized and attached to type uint256, making all values of type uint256 in the LiquidationBranch smart contract have access to functions like toUint128().

`event LogLiquidateAccount(
        address indexed keeper,
        uint128 indexed tradingAccountId,
        uint256 amountOfOpenPositions,
        uint256 requiredMaintenanceMarginUsd,
        int256 marginBalanceUsd,
        uint256 liquidatedCollateralUsd,
        uint128 liquidationFeeUsd
    );`
- This is an event called LogLiquidateAccount and has topics “keeper” which is of type address and indexed, “tradingAccountId” an indexed uint128 type, “amountOfOpenPositions” type uint256, “requiredMaintenanceMarginUsd” type uint256, “marginBalanceUsd” type int256,  “liquidatedCollateralUsd” type uint256, “liquidationFeeUsd” type uint128.

`function checkLiquidatableAccounts(
     uint256 lowerBound,
     uint256 upperBound
)
     external
     view
     returns (uint128[] memory liquidatableAccountsIds)
{`
- This is declaring a view function checkLiquidatableAccounts which takes parameters lowerBound and upperBound, both of type uint256. This function has an external visibility and returns liquidatableAccountsIds which is an array of type uint128.

`  liquidatableAccountsIds = new uint128[](upperBound – lowerBound);`
- This initializes the return variable and determines the size of the array.

`if (liquidatableAccountsIds.length == 0) return liquidatableAccountsIds;`
- Checks the length of  liquidatableAccountsIds and returns it if there’s no id to process. 

` GlobalConfiguration.Data storage globalConfiguration = GlobalConfiguration.load();`
- Creates an instance of the Data struct from  GlobalConfiguration in storage called globalConfiguration and fetches the storage slot for the global configs using load().

` uint256 cachedAccountsIdsWithActivePositionsLength =         globalConfiguration.accountsIdsWithActivePositions.length();`
- Declares a variable  cachedAccountsIdsWithActivePositionsLength and is used to store the current length of active positions in the account ids which is gotten from  globalConfiguration.

` for (uint256 i = lowerBound; i < upperBound; i++) {`
- Start a for loop to iterate over active accounts with the lowerBound and upperBound as the range.

`if (i >= cachedAccountsIdsWithActivePositionsLength) break;`
- Checks if “i” is greater than the length of active account ids and breaks if true.

` uint128 tradingAccountId = uint128(globalConfiguration.accountsIdsWithActivePositions.at(i));`
- Create a uint128 value called  tradingAccountId and assign it the trading account id of the current active account. To make this possible in solidity, the unit256 type was converted to uint128.

`TradingAccount.Data storage tradingAccount = TradingAccount.loadExisting(tradingAccountId);`
- Creates an instance of the Data struct from TradingAccount in storage called tradingAccount and loads the active account leaf (data and functions) using the loadExisting function from  TradingAccount library.

`(, UD60x18 requiredMaintenanceMarginUsdX18, SD59x18 accountTotalUnrealizedPnlUsdX18) = tradingAccount.getAccountMarginRequirementUsdAndUnrealizedPnlUsd(0, SD59x18_ZERO);`
- Get that account's required maintenance margin & unrealized PNL, by calling  getAccountMarginRequirementUsdAndUnrealizedPnlUsd() on tradingAccount which returns a tuple. The tuple is destructured to get just the values we need which are requiredMaintenanceMarginUsdX18 of type UD60x18 and accountTotalUnrealizedPnlUsdX18 of type  SD59x18.

`SD59x18 marginBalanceUsdX18` = tradingAccount.getMarginBalanceUsd(accountTotalUnrealizedPnlUsdX18);`
- Get the current active account’s margin balance by calling  getMarginBalanceUsd on  tradingAccount and storing it in a newly created variable of type SD59x18 called  marginBalanceUsdX18.

`if (TradingAccount.isLiquidatable(requiredMaintenanceMarginUsdX18, marginBalanceUsdX18)) {
	liquidatableAccountsIds[i] = tradingAccountId;
}`
- Check if requiredMargin > marginBalance and if it is, set the account for liquidation by adding that account id to the  liquidatableAccountsIds array.

`struct LiquidationContext {
        UD60x18 liquidationFeeUsdX18;
        uint128 tradingAccountId;
        SD59x18 marginBalanceUsdX18;
        UD60x18 liquidatedCollateralUsdX18;
        uint256[] activeMarketsIds;
        uint128 marketId;
        SD59x18 oldPositionSizeX18;
        SD59x18 liquidationSizeX18;
        UD60x18 markPriceX18;
        SD59x18 fundingRateX18;
        SD59x18 fundingFeePerUnitX18;
        UD60x18 newOpenInterestX18;
        SD59x18 newSkewX18;
}`
- Declaration of struct LiquidationContext which contains 4 values of type UD60x18 called liquidationFeeUsdX18, liquidatedCollateralUsdX18, markPriceX18 and newOpenInterestX18, 6 values of type SD59x18 called marginBalanceUsdX18, oldPositionSizeX18, liquidationSizeX18, fundingRateX18, fundingFeePerUnitX18, newSkewX18, 2 variables of type uint128 called tradingAccountId and marketId and an array of uint256 called activeMarketsIds. All these properties, describe a liquidation context.

`function liquidateAccounts(uint128[] calldata accountsIds) external {`
- Declaration of function liquidateAccounts which takes a parameter of array of uint128 called  accountsIds and has an external visibility. The accountsIds are ids of accounts to be liquidated.

`if (accountsIds.length == 0) return;`
- Check if there are any accountsIds and return if there are none.

`GlobalConfiguration.Data storage globalConfiguration = GlobalConfiguration.load();`
- Creates an instance of the Data struct from  GlobalConfiguration in storage called globalConfiguration and fetches the storage slot for the global configs using load().

`if (!globalConfiguration.isLiquidatorEnabled[msg.sender]) {
            revert Errors.LiquidatorNotRegistered(msg.sender);
   }`
- Checks if the msg.sender is an authorized by liquidator by checking the isLiquidatorEnabled mapping from GlobalConfiguration instant. If the msg.sender is not authorized, revert with custom error LiquidatorNotRegistered(). 

`LiquidationContext memory ctx;`
- Create an instance of LiquidationContext struct as variable called ctx and store it in the memory. 

`ctx.liquidationFeeUsdX18 = ud60x18(globalConfiguration.liquidationFeeUsdX18);`
- Get liquidation fee from global config, converted to type ud60x18 and set it as the value of the liquidationFeeUsdX18 in ctx.  

` for (uint256 i; i < accountsIds.length; i++) {`
- Start a loop to iterate over the array of accountsIds to liquidate. 

`ctx.tradingAccountId = accountsIds[i];`
- Store the current accountId to be liquidated as the tradingAccountId of ctx.

`if (ctx.tradingAccountId == 0) continue;`
- Check if the tradingAccountId is 0 then skip the loop execution on that accountId and if it’s not 0, then continue with the loop. This is commonly called a sanity check.

`TradingAccount.Data storage tradingAccount = TradingAccount.loadExisting(ctx.tradingAccountId);`
- Load the account’s leaf (data and functions) by creating an instance of the Data struct with the tradingAccountId from TradingAccount by calling the loadExisting function from TradingAccount.

`(, UD60x18 requiredMaintenanceMarginUsdX18, SD59x18 accountTotalUnrealizedPnlUsdX18) = tradingAccount.getAccountMarginRequirementUsdAndUnrealizedPnlUsd(0, SD59x18_ZERO);`
- Get that account's required maintenance margin & unrealized PNL, by calling  getAccountMarginRequirementUsdAndUnrealizedPnlUsd() on tradingAccount which returns a tuple. The tuple is destructured to get just the values we need which are requiredMaintenanceMarginUsdX18 of type UD60x18 and accountTotalUnrealizedPnlUsdX18 of type SD59x18.

`ctx.marginBalanceUsdX18 = tradingAccount.getMarginBalanceUsd(accountTotalUnrealizedPnlUsdX18);`
- Get the marginBalanceUsd from tradingAccount by calling getMarginBalanceUsd() with accountTotalUnrealizedPnlUsdX18 passed as the argument. The value returned is stored to  marginBalanceUsdX18 of the ctx struct.

` if (!TradingAccount.isLiquidatable(requiredMaintenanceMarginUsdX18, ctx.marginBalanceUsdX18)) {
                continue;
}`
- Check if this account is liquidable by calling isLiquidatable function with  requiredMaintenanceMarginUsdX18 and ctx.marginBalanceUsdX18 as argument from TradingAccount library. This function returns the value requiredMaintenanceMarginUsdX18 > ctx.marginBalanceUsdX18. If the account is not liquidable, it skips this account in the loop but if it liquidable, the execution continues.

`ctx.liquidatedCollateralUsdX18 = tradingAccount.deductAccountMargin({
         	feeRecipients: FeeRecipients.Data({
         		marginCollateralRecipient: globalConfiguration.marginCollateralRecipient,
		orderFeeRecipient: address(0),
            	settlementFeeRecipient: globalConfiguration.liquidationFeeRecipient
 	}),
                pnlUsdX18: requiredMaintenanceMarginUsdX18,
                orderFeeUsdX18: UD60x18_ZERO,
                settlementFeeUsdX18: ctx.liquidationFeeUsdX18
});`
- This calculates the liquidatedCollateralUsdX18 by calling deductAccountMargin from the TradingAccount library. DeductAccountMargin() takes a struct of type FeeRecipients.Data, the pnlUsdX18, settlementFeeUsdX18 and orderFeeUsdX18 all of type UD60x18, and it returns marginDeductedUsdX18 which is of type UD60x18. In this call, the FeeRecipients.Data is created from globalConfiguration.marginCollateralRecipient as the marginCollateralRecipient, address(0) as the orderFeeRecipient and globalConfiguration.liquidationFeeRecipient as  settlementFeeRecipient. While requiredMaintenanceMarginUsdX18 is passed as the pnlUsdX18,  UD60x18_ZERO as the orderFeeUsdX18 and ctx.liquidationFeeUsdX18 as the settlementFeeUsdX18. The value returned is stored in the liquidatedCollateralUsdX18 property of ctx.

`MarketOrder.load(ctx.tradingAccountId).clear();`
- Clear() function is called from the MarketOrder library to clear the pending orders for the account being liquidated. To call clear(), MarketOrder is first initialized by calling load() on the MarketOrder library and passing ctx.tradingAccountId as the argument.

`ctx.activeMarketsIds = tradingAccount.activeMarketsIds.values();`
- Copy and store the active market ids for the account being liquidated to the activeMarketsIds property of ctx. This is done by calling activeMarketsIds.values() from the TradingAccount library.

`for (uint256 j; j < ctx.activeMarketsIds.length; j++) {`
- Start another loop, iterating over the copy of active market ids in ctx.

`ctx.marketId = ctx.activeMarketsIds[j].toUint128();`
- Load and store the id of the active market into the marketId property of ctx. This is done by checking value in the array activeMarketsIds in ctx at index j and downcasting the value from uint256 to uint128 with toUint128() from the SafeCast library.

`PerpMarket.Data storage perpMarket = PerpMarket.load(ctx.marketId);`
- Initialise Data struct from PerpMarket into the storage by calling the load function from  PerpMarket and passing the current market id as an argument.

`Position.Data storage position = Position.load(ctx.tradingAccountId, ctx.marketId);`
- Initialise Data struct from Position into the storage by calling the load function from Position and passing the current market id and the tradingAccountId from ctx as arguments.

`ctx.oldPositionSizeX18 = sd59x18(position.size);`
- Cast the open position size in the current market by using sd59x18 and store the value to the oldPositionSizeX18 property of ctx.

`ctx.liquidationSizeX18 = -ctx.oldPositionSizeX18;`
- Get the negative value of the previously stored position size in ctx.oldPositionSizeX18 and store the negative value to the  liquidationSizeX18 property of ctx. This is to prepare to close the position.

`ctx.markPriceX18 = perpMarket.getMarkPrice(ctx.liquidationSizeX18, perpMarket.getIndexPrice());`
- Get the price impact of the position to be close from PerpMarket by calling  getMarkPrice from the PerpMarket library and passing ctx.liquidationSizeX18 and perpMarket.getIndexPrice() as parameters. GetMarkPrice() takes in 2 arguments of type SD59x18 and UD60x18 and returns the markPrice which is of type UD60x18. The returned value is stored in the markPriceX18 property of ctx.

`ctx.fundingRateX18 = perpMarket.getCurrentFundingRate();`
- Get current funding rate by calling getCurrentFundingRate from the PerpMarket library.  GetCurrentFundingRate() returns current funding rate of the current market as a type SD59x18. This value is stored to the fundingRateX18 property of ctx.

`ctx.fundingFeePerUnitX18 = perpMarket.getNextFundingFeePerUnit(ctx.fundingRateX18, ctx.markPriceX18);`
- Get the current funding fee per unit for the current market and store it to the  fundingFeePerUnitX18 property of ctx. This is done by calling  getNextFundingFeePerUnit from the PerpMarket library, passing the current funding rate stored in  fundingRateX18 of ctx and mark price stored in  markPriceX18 of ctx.  GetNextFundingFeePerUnit() returns a type of SD59x18, which will be stored in fundingFeePerUnitX18 property of ctx.

`perpMarket.updateFunding(ctx.fundingRateX18, ctx.fundingFeePerUnitX18);`
- Update the funding rates for the current market by calling the updateFunding function from PerpMarket library and passing fundingRateX18 from ctx and fundingFeePerUnitX18 from ctx as arguments.

`position.clear();`
- Reset the position by calling the clear function from the Position library.

`tradingAccount.updateActiveMarkets(ctx.marketId, ctx.oldPositionSizeX18, SD59x18_ZERO);`
- Update the account’s active markets by calling updateActiveMarkets function from the TradingAccount library. The current market id stored in ctx.marketId, the position size stored in ctx.oldPositionSizeX18 and SD59x18_ZERO as the arguments.  UpdateActiveMarkets() uses remove() from the EnumerableSet library to update the markets.

`perpMarket.updateOpenInterest(ctx.newOpenInterestX18, ctx.newSkewX18);`
- Update the open interest by calling updateOpenInterest from the PerpMarket library. This function takes  ctx.newOpenInterestX18, ctx.newSkewX18 as arguments. It also updates skew.

`emit LogLiquidateAccount(
    	msg.sender,
            ctx.tradingAccountId,
            ctx.activeMarketsIds.length,
            requiredMaintenanceMarginUsdX18.intoUint256(),
       	    ctx.marginBalanceUsdX18.intoInt256(),
            ctx.liquidatedCollateralUsdX18.intoUint256(),
            ctx.liquidationFeeUsdX18.intoUint128()
   	);`
- Event LogLiquidateAccount is emitted after the account is liquidated. Msg.sender, ctx.tradingAccountId, ctx.activeMarketsIds.length, requiredMaintenanceMarginUsdX18, ctx.marginBalanceUsdX18, ctx.liquidatedCollateralUsdX18, ctx.liquidationFeeUsdX18 are passed as arguments for the topics being emitted in the event.
